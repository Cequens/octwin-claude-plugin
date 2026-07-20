# `rbac` — pack-declared role-based access control

**Business:** a workspace admin decides *what each person may do* — "Layla works the Western queue and can approve refunds; Hassan only reviews captain applications; the auditor sees everything and touches nothing." **Technical:** a pack declares structured domain **roles** in `<packRoot>/roles.yaml` (a key, a human `description:`, and permission **grants**), a tenant admin **assigns** users/teams to them per project (Console → Team → **Roles**), and the platform **enforces** them at a small, auditable set of service choke points.

> Training guide (Artifact): <https://claude.ai/code/artifact/81f119d7-1f0c-4ac4-aecf-59d7d79367a2>
> — source [`docs/artifacts/rbac-module-guide.html`](../../../docs/artifacts/rbac-module-guide.html). Keep it current with this module.

The sibling of [`entitlements`](../entitlements/README.md): **entitlements gate FEATURES per tenant; rbac gates ACTIONS per principal.** Both sit low in the layer-DAG (rbac is rank 9, directly above `identity`) precisely so every module above — `xrm`, `worklist`, `casework`, `automation`, `journey`, and the routes — can import it **downward** to enforce.

---

## The model in one page

A grant answers three questions: **what** (resource), **which action** (verb), **how far** (scope).

| Axis | Values | Who defines it |
|---|---|---|
| **Resource** | `<domain>[.<subject>]` — `record.case`, `record.lead`, `automation.job`, `journey`, with wildcards `record.*` / `*` | The **domain** is registered by the module (`RbacResourceCatalog`); the **subject** is the pack's own entity |
| **Verb** | `RBAC_VERBS` — `view · create · update · note · transition · assign · act · archive` | Fixed platform taxonomy (`contracts/spec.ts`) |
| **Scope** | `RBAC_SCOPES` — `own · team · queue · all` (a list = union) | Fixed; which scopes a domain supports is in its descriptor |
| **Action** | an optional allowlist on `act` — `{ scope: queue, actions: [approve_refund] }` | The pack's declared worklist actions / case dispositions / module ops (`run`, `send`) |

**`act` is the extension point, not the verb list.** A module-specific operation (approve a refund, run a sweep, send a campaign) is an `act` **action key**, validated against the resource catalog at boot. The verb set stays fixed and module-agnostic — which is what keeps the console, the schema, and the storage from learning any pack's vocabulary.

### How the tenant role and the pack roles layer (additive)

| Tenant role | RBAC authority |
|---|---|
| platform operator / `owner` / `admin` / break-glass token | `seesAll` — bypass (this is the rule the retired `resolveWorkAccess` used, so operators keep working with zero assignments) |
| `member` | the **union** of the grants of every role assigned to them **or to a team they belong to**. **No assignment = no work-layer access** (deny-by-default) |
| `viewer` | the same grants, then `readOnly: true` **masks off every verb except `view`** — this is what finally makes the long-dormant `viewer` rank mean something |

Assignments live in `role_assignments` (migration 077). The role **catalog** does not: it lives in `RbacRegistry`, resolved from the pack's YAML + the platform-standard trio. That is why the table carries **no FK on `role_key`** — validity is enforced at *write* (the route checks the registry), and a key that later disappears (a pack edit) is skipped at *read* with a warning and flagged `stale` in the console. A pack edit can never 500 an operator.

### Queue binding happens at ASSIGNMENT time

A `queue`-scoped grant reaches the queues bound on the assignment (`role_assignments.queues`), falling back to the role's declared default `queues:`. One `regional_agent` role therefore serves every region — bind the Western team's assignment to `ops_western`, the Central team's to `ops_central`. (This subsumes the retired `worklist_members` table, which was a bare (queue, principal) grant with no verbs and no UI — dropped in 077.)

---

## Enforcement — the audit table

Enforcement is **not** scattered. Services take an optional `access?: RbacAccess`; **absent = a trusted internal caller** (agent primitives, automation's `actor:'system'` sweeps, provisioning/seeds, and nested service composition), which is unrestricted. Operator-facing routes **always** pass it, and the **outermost call authorizes once** — the *single-gate rule* (so `recordDecision` → `transition` doesn't double-gate).

`FULL_RBAC` is the one intentional bypass constant: **`grep FULL_RBAC` enumerates every place enforcement is deliberately skipped.**

| Verb | Choke point (the gate) |
|---|---|
| `view` (point) | `assertCanForRecord(access, 'view', …)` — `routes/cases.ts` get-one, `routes/xrm.ts` get-one |
| `view` (lists) | `scopeFor(...)` → `ListRecordsFilter.workScope` / `ListCasesFilter.workScope` / `ListItemsFilter.workScope`, rendered by the **one** predicate renderer `renderWorkScopeSql` |
| `create` | `saveRecord` (create branch) → `assertCanCreate` |
| `update` | `saveRecord` (update branch), `linkRecord` |
| `note` | `addRecordNote`, casework `addNote` |
| `transition` | `transitionStage`, casework `transition`, `completeTask` |
| `assign` | worklist `assign`, casework `assignCase` |
| `act` | `applyRecordAction`, casework `recordDecision` + `previewDecision` (+ the action allowlist); `automation` job run/pause/resume + campaign send (route-level `assertCan`) |
| `archive` | `setRecordArchived` |

**The completeness of this table IS the security property.** It is verified end-to-end by [`src/routes/rbac-enforcement.test.ts`](../../routes/rbac-enforcement.test.ts) — a matrix of five principals (owner · queue-scoped agent · differently-scoped agent · viewer · member-with-no-role) against the real Fastify handlers. A new record/work endpoint that forgets to pass `access` shows up there as a row a scoped member can still reach.

---

## Two extension points — know which is yours

**Pack authors never touch TypeScript for RBAC.** A pack's whole surface is `roles.yaml`, and *extending the
grantable set* is also YAML: declaring an entity in `xrm.yaml`, an action in `worklist.yaml`, or a disposition in
`cases.yaml` makes `record.<entity>` / that action / that queue grantable automatically, because the `record`
domain descriptor reads the pack registries generically (`subjects(packId) → XrmRegistry.listEntities(packId)`).
No registration step.

The recipe below is **platform engineering** — run once when a whole new platform *capability layer* is built
(the way `record`/`automation`/`journey` each did), in `src/platform/`, never in `src/packs/`. It's TypeScript
because a descriptor is **behaviour, not data**: `subjects`/`actions` are functions that compute the securable set
from live registries per pack — the same declarative-YAML / TS-escape-hatch line the platform draws for
`definePrimitive`.

## Adding a securable LAYER (platform-engineer recipe)

The engine knows **domains**, not modules. To bring a new platform module (say a `billing` layer) under RBAC:

1. **Register a domain descriptor** — add it to `registerPlatformRbacDomains()` in [`pack/rbac-domains.ts`](../pack/rbac-domains.ts) (the one composition point where every registry is in scope):

   ```ts
   RbacResourceCatalog.register({
     domain:        'billing',
     allowedScopes: ['all'],                      // no containment concept → `all` only
     subjects:      () => ['invoice', 'refund'],
     actions:       (_pack, subject) => (subject === 'refund' ? ['issue'] : null),
   })
   ```

2. **Gate its choke points** — `can(access, verb, { resource: 'billing.invoice' })` / `assertCan(...)` in the service or route, and `scopeFor(...)` on its lists.

3. That's it. The YAML schema, boot validation, `role_assignments`, the resolver, and the console Roles tab **all come free** — none of them knows a module name. Once the layer exists, a **pack** author grants over it in the usual YAML — `billing.refund: { act: { scope: all, actions: [issue] } }` — and boot validation checks every word against the descriptor.

---

## Files

| File | Role |
|---|---|
| `contracts/spec.ts` | The `roles.yaml` Zod schema + the fixed `RBAC_VERBS` / `RBAC_SCOPES` |
| `resource-catalog.ts` | `RbacResourceCatalog` — each securable module registers its domain descriptor |
| `standard-roles.ts` | `agent` / `supervisor` / `read_only`, merged **under** every pack's roles (the `CASE_STANDARD_ACTIONS` preset-merge pattern) |
| `resolver.ts` | Validation (taxonomy + cross-registry), `buildPackRoles`, the console projection |
| `registry.ts` | `RbacRegistry` — per-pack role catalog (registered for **every** pack, even without a `roles.yaml`) |
| `assignments-store.ts` | `role_assignments` CRUD (migration 077) |
| `access.ts` | `resolveRbacAccess` — `Principal` → `RbacAccess` (the tenant-role layering lives here) |
| `evaluate.ts` | `can` / `assertCan` / `assertCanForRecord` / `scopeFor` + `FULL_RBAC` |
| `sql.ts` | `renderWorkScopeSql` — the one read-scoping predicate |

**Related:** [`docs/26-identity-auth-rbac.md`](../../../docs/26-identity-auth-rbac.md) (the canonical auth model) · [`docs/12-pack-authoring.md`](../../../docs/12-pack-authoring.md) (`roles.yaml` authoring) · [`identity/README.md`](../identity/README.md) (principals, teams, principal-refs) · [`worklist/README.md`](../worklist/README.md) (queues + assignment).
