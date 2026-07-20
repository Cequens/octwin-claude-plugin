# Pack authoring

**Status: AUTHORITATIVE.** Source files cited here are the canonical
truth; this doc summarises the contract for pack authors and points at
the right reference packs to copy from.

## The rule

**Pack code imports platform APIs through the `#sdk` alias ([`platform_sdk.ts`](../src/platform/platform_sdk.ts)) only.**

This is lint-enforced (the `packs` rule in `eslint.config.mjs`) with **zero exemptions**. Any other platform specifier — `#platform/...` or a deep relative like `from '../../../platform/flow-runtime/foo.js'` — is a contract violation that fails `npm run lint`. If the SDK barrel doesn't export what you need, that's a signal — either:
- The thing you want is genuinely platform-internal and shouldn't be in pack code (find another way), or
- The SDK is missing an export it should have (add it to `platform_sdk.ts`, or open a discussion).

The SDK is the contract. Anything not exported there is platform-internal and may change without notice. (The boundary model — layers, barrels, the `#`-aliases — is specified in [`07-module-architecture-and-imports.md`](07-module-architecture-and-imports.md).)

## What's in the SDK

[`src/platform/platform_sdk.ts`](../src/platform/platform_sdk.ts) is the source of truth — read it. The surface is deliberately small:

| Tier | Purpose | Exports |
|---|---|---|
| 1. Flow authoring (escape hatch) | Pack-primitive factory + output-port constructor (+ types) — for the rare logic the YAML expr grammar can't express; **not** the default authoring path (see below) | `definePrimitive`, `port`, `FlowExecContext`, `FlowPrimitive` |
| 2. Observability | Traced logger | `log` |
| 3. Channel/adapter types | Channel + messaging contract types (cross-user sends use `ctx.adapters.messaging`) | `Channel`, `MessagingAdapter` |
| 4. Pack runtime + env | Pack-runtime ALS accessor + env read (use `ctx.env` inside flows; `getPackEnv` only at module-level) | `requirePackRuntime`, `getPackEnv`, `loadPackEnv`, `PackEnvMap` |
| 5. Child-process DI | The contract the platform child entry injects into a pack's route registrar | `PackChildAdapters` |

Each export's doc-comment in `platform_sdk.ts` is the spec — describes the contract, not just the type. Note there is no `getChannelImpl` export: cross-user / cross-conversation sends go through `ctx.adapters.messaging` (flow primitives / handlers) or the platform-injected messaging factory in child routes (below) — never by importing channel modules.

### Platform-builtin primitives (no registration needed)

A small set of primitives are auto-injected into every flow's primitives map by the platform — packs reference them by name in YAML without any per-pack registration:

| Primitive | Purpose |
|---|---|
| `do: notify` | Channel-agnostic cross-user delivery (recipient ≠ inbound sender). Persists to `notifications`, supports pre-turn drain + echo_delivery. |
| `do: send_message` | Same-conversation outbound mid-flow (extra messages outside `render:`). |
| `do: get_media` / `set_media_metadata` | Read/write `media_assets` rows. |
| `do: upsert_platform_contact` / `resolve_contact_by_metadata` | Pack-domain overlay onto platform contacts; lookup by metadata key. |

Full reference: [`src/platform/flow-runtime/schemas/context.md`](../src/platform/flow-runtime/schemas/context.md) §11b. Machine-readable catalog: [`dist/octwin-platform-kb/platform-primitives.json`](../dist/octwin-platform-kb/platform-primitives.json). (The legacy `inject_memory_note` primitive was retired in Phase 15 — set a top-level `memory_note` on your `ToolResponse` envelope instead.)

Pack primitives shadow platform builtins on name conflict (intentional override path for tests; never used in production). Avoid declaring a pack primitive named `notify` / `send_message` / `get_media` unless you're deliberately overriding the platform behavior.

## Conventions

### Prefer an inline `item_template` over a `$map(rows, mapper)` partial

A render's `item_template` evaluates a **full per-row expression** (`$item` bound to the
raw row): `$enum_label`, `$format_money`, ternaries, `$coalesce`, conditional fields, and
per-row `on_select.with` all work. So a single-use mapper partial that just shapes rows
for ONE render is redundant — pass the **raw** rows as `items:` and compute the view-model
inline in `item_template`, deleting both the partial and the `assign items: $map(...)`
step. Reserve a mapper/view-model partial only when (a) the shaped rows are used by **2+
renders**, (b) they also feed the `data:` envelope, or (c) they normalise *heterogeneous*
sources into the common shape a **shared** render step consumes (e.g. the `*_option`
mappers feeding `render_field_picker`). For auto-sectioning use `group_by: <field>` on
`list_picker` (see [13a §2](13a-dsl-reference.md)) instead of
`$map($entries($group_by(...)), wrapper)`.

### Author in pure YAML; pack TS primitives are an escape hatch

**Packs are authored in pure YAML.** Flows compose the platform-builtin primitives (`db_*` / `db_rpc` / `embed_text` / `analyze_image` / `storage_*` / `notify` / `log_values` / …) with the expr-builtin library (`$coalesce` / `$trim` / `$get` / `$lines` / `$format_*` / ternaries / `{$expr}` interpolation) and the flow constructs (`collect:` / `dispatch:` / `if` / `assign` / `render`). Every shipped pack is pure-YAML — **zero pack TypeScript primitives**:

- [`real-estate`](../src/packs/real-estate/) — the full domain pack. Even Arabic multi-line card bodies (the historical "needs a string-builder primitive" case) are YAML, via `$lines` + `$get` enum maps + the `listing_view` / `compound_info_view` partials.
- [`showcase`](../src/packs/showcase/) + [`starter-kit`](../src/packs/starter-kit/) — capability demos and the starter template, also fully pure-YAML.

**A pack TS primitive (`definePrimitive`) is the escape hatch** for the rare piece of logic the expr grammar genuinely cannot express — reach for one only after confirming a `do: <platform builtin>` + exprs can't do the job. It is NOT the default. When you do write one:

- **It produces DATA, never a render intent.** No `render_intent`, no `item_template` lambdas, no picker `on_select` rows in TS — return view-model data (`{ title, description, … }` or a plain string) and let the YAML `render` node compose the intent. The runtime's UI-shape detector rejects a primitive that returns a `UiHint`.
- It reaches infra through the injected `ctx` (db / messaging / storage / env), never a global accessor (see *Pack is tenant-blind*).

### Manifest path-refs for handlers

Pack handler wiring lives in `manifest.yaml`, not in TS boot files. Each path-ref is `'./path/to/file.js#namedExport'`:

```yaml
contact_resolver: './services/supabase.service.js#getUserByContactHandle'

channel_hooks:
  resolve_user:     './handlers/channel-hooks.js#resolveUser'
  get_user:         './services/supabase.service.js#getUserByContactHandle'
  process_image:    './handlers/channel-hooks.js#processImage'
  process_document: './handlers/channel-hooks.js#processDocument'

conversation:
  get_user:   './repositories/user.repo.js#getUserById'
  to_contact: './handlers/channel-hooks.js#toContact'

status_handler:
  delivered: './domain/leads/relay-delivery.js#handleRelayDelivery'

agents:
  - id: broker
    hooks:
      collect_pre_turn_notes: './domain/leads/relay-briefings.js#collectPendingNotes'
      append_token_usage:     './services/supabase.service.js#appendTokenUsage'
```

The platform's [`pack/handlers/bootstrap.ts`](../src/platform/pack/handlers/bootstrap.ts) leaf-entry dynamic-imports each ref at boot and threads the resolved functions through `platformChannelHooks(...)` + `createWithPackConversation(...)` + `platformAgentHooks(...)` internally. Pack `handlers/channel-hooks.ts` is pure domain — zero platform imports, zero `platformChannelHooks(...)` wrapper call.

### Pack is tenant-blind

Don't import `currentTenantId` / `currentProjectId` / `withTenant` or `getAdapters()`. The platform owns request-scoping end-to-end:
- **DB access** — use the pack's `db` proxy (`src/packs/<id>/db/client.ts`) which delegates to `requirePackRuntime().adapters.dataStore.getClientSync()`. The adapter is always schema-scoped to the active install — routes use `IpcDataStoreAdapter` (IPC bridge to main thread), events/flows use `PackScopedDataStoreAdapter` (main process, schema-scoped wrapper). Pack code writes `db.from('leads').select(...)` identically everywhere; schema scoping is automatic.
- **Messaging** — use `ctx.adapters.messaging` (flow primitive or handler context) for cross-conversation sends; in child-process HTTP routes use the platform-injected messaging factory (see "Pack HTTP routes" below). Never import channel modules directly.

The cap-aware UI decisions (list maxRows etc.) also belong to the platform — render intents truncate at render time. Don't read `ctx.vars.caps` to make sizing decisions in pack code.

### Bare schema + descriptive field names

Tool input fields without `.describe()` are FINE when the field name is self-evident (`bedrooms`, `area_m2`, `photos`). Reserve `.describe()` for non-obvious semantics or domain rules only the LLM can enforce.

### Disambiguate opaque-id inputs from free-text

A tool that accepts both an **opaque id** (a UUID/key the agent only ever obtains from a prior tool result or a picker selection) *and* a **free-text** field (whatever the user typed) is a misfile trap: the agent will sometimes drop typed text into the id field, and that fails hard at the data layer (e.g. a UUID cast error), not gracefully. The field *name* alone won't prevent it — this is exactly the "non-obvious rule" case for `.describe()`. Spell out both classes so the routing is unambiguous:

- **id field** — "an id, **only** from a previous tool result / picker — never text the user typed."
- **free-text field** — "anything the user typed (a name, a category word, a phrase) goes here."

Fix it at the agent's *contract* (the description) — do **not** paper over the misfile with a defensive cast/guard inside the flow. Reinforce the same split in the agent's instructions where the tool is introduced.

### Describe only what the logic delivers

A field's `.describe()` is a promise to the agent. If you tell it a field accepts a kind of value, the flow/query behind that field must actually handle it — telling the agent "typed category names go in the free-text field" is a lie if the search only matches one column, and the user gets empty results. Make the description honest, or extend the logic to match it; never advertise an input capability the data layer doesn't deliver.

### Brief tool descriptions

Descriptions are intent + domain-quirks only. No flow narration, no channel mechanics, no per-step what-happens-next — the runtime channels (`instructions`, `memory_note`, suppression, `workflow_run_id` round-trip) keep the agent aware turn-by-turn.

**Mark interactive tools.** If a tool drives its own multi-turn collection over the channel (forms, picker cascades, disambiguation, preview-and-confirm — anything where the *tool* prompts the user for fields), open its description with the literal marker **`[interactive Tool]`**. The platform agent-protocol (§2) teaches every agent that such a tool owns collection: the agent calls it on intent with only the just-stated fields and never interviews the user in free text first — on every invocation, including repeats. Tap-driven view/action tools (a menu, a "my listings" list) don't collect schema fields and should NOT carry the marker.

### Conditional values in render fields

Three forms compose cleanly inside any scalar render field (header, body, footer, cta, button title, etc.). Pick the form that reads best at the call site:

**1. Inline ternary** — for short two-way branches that fit on one line:

```yaml
footer: '$view.is_live ? $t("listings.manage_footer_live") : $t("listings.manage_footer_archived")'
```

The expression evaluator supports `cond ? a : b` natively (jsep `ConditionalExpression`). Use when both branches are short single values.

**2. `{if, then, else}` object** — for multi-line or structurally different branches:

```yaml
sections:
  - items:
      - if: '$view.is_live'
        then:
          title: '{$t("listings.row_unlist_title")}'
          on_select: { invoke: $self, with: { action: unlist } }
        else:
          title: '{$t("listings.row_relist_title")}'
          on_select: { invoke: $self, with: { action: relist } }
```

The render-resolver walks `{if, then, else}` objects recursively (see [`render-resolver.ts`](../src/platform/flow-runtime/interpreter/render-resolver.ts)). Use for picking between *whole render fragments* (rows, buttons, sections), not just scalar strings.

**3. `{$expr}` interpolation in plain strings** — for mid-string substitutions:

```yaml
body: '🛠️ {$view.reference} — {$view.location}'
memory_note: 'Showed manage card for {$view.reference} [{$view.status}]'
```

Use when text + expressions interleave. Brace-balanced — nested object literals inside `{...}` are scanned correctly.

**Anti-pattern:** pre-resolving conditional values into an `assign:` step purely to keep the render field a single chip. The three forms above cover this without the extra step. Use `assign:` only when the resolved value is referenced *multiple times*.

### Phase-1 runtime ergonomics

These four shortcuts are baked into the runtime — leveraging them keeps flows terse:

- **`$input` is this turn's delta; `$state` is the run's accumulator.** A resume/tap `with` payload becomes THIS turn's `$input`, and the engine folds it into `$state` (`state = { ...state, ...input }`) before any node runs. Read `$state.x` for any value that must survive a `pause`/`collect` suspend (a `collect` node accumulates its fields into `$state`); read `$input.x` only for genuinely this-turn signals. Decisions ride the separate `$control` channel (`on_select.control`), never `$input`/`$state`. (The retired `$resume` auto-merge was the pre-fold staging of `$input`.) **A resume/tap `with` payload should carry ONLY the value(s) the tap contributes this turn — don't re-pass context already in `$state`** (e.g. a slot-picker resume sends `{ new_slot_start, slot_minutes }`, not `{ appointment_id, … }`; read `$state.appointment_id` after resume). See [`manage-appointments.flow.yaml`](../src/packs/clinic/flows/tools/manage-appointments.flow.yaml) `show_reschedule_picker`.
- **Default `bind:` is the primitive name.** `do: get_property` with no `bind:` binds the result to `$get_property`. Set `bind:` explicitly only when a shorter name reads better at the call site.
- **`outputs:` ↔ `on_null:` are mutually exclusive.** Use ports for primitives that branch (`fail / gap / ready / not_found / …`); `on_null:` is the legacy escape for primitives that just return null.
- **`memory_injection` auto-rule.** Suppression-mode renders (no agent narration) auto-persist their `memory_note`. Set `narrate: true` only when the agent should speak after the card lands.

### Phase-2 conventions: port-defaults table

Common port names get behavioral defaults (path, mode, canonical handler) from a central table:

| Port name | Canonical handler (`outputs: { <port>: [] }`) | Mode |
|---|---|---|
| `applied` / `ok` | `end: applied` | suppression |
| `aborted` | render `text_card($<bind>.reason)` + `end: aborted` | suppression |
| `failed` / `fail` | render `text_card($<bind>.reason)` + `end: failed` | suppression |
| `not_found` / `denied` / `empty` | path-only default (canonical render TBD in Phase 3) | suppression |
| `ready` | path-only default (typically a `goto`) | suppression |
| `gap` | path-only (mode left to call-site: render-less → complete; custom-render → suppression) | — |

Empty-array sentinel for canonical: when a primitive declares an `outputs: { failed: [] }` entry and the port has a canonical handler, the runtime synthesizes the standard sequence. Override by supplying a non-empty body in YAML.

Authoritative source: [`port-defaults.ts`](../src/platform/flow-runtime/port-defaults.ts). Importable from the SDK barrel as `getPortDefaults(name)`, `CANONICAL_PORT_NAMES`, `PortDefaults`.

### Flow input schema drives tool catalogue + gap-collection

The flow's top-level `input:` block is the single source of truth for the tool's input contract:

```yaml
input:
  property_id:      { type: string, optional: true,  describe: 'Property UUID or unit reference.' }
  action:           { type: enum,   enum: [contact, offer, cancel], optional: true }
  offer_amount_egp: { type: number, positive: true,  optional: true, describe: 'Required when action is "offer".' }
  workflow_run_id:  { type: string, optional: true }
```

The platform generates the Mastra tool's Zod schema from this block via `inputFieldsToZod()`. Three things flow from this:

1. **LLM tool catalogue** — what the agent sees: field names, types, `describe()` strings, required vs optional, enums.
2. **Runtime validation** — every invocation (LLM-driven and direct-tap) re-validates against the same schema.
3. **Per-project overrides** — when a project customizes a flow's input shape (e.g. requiring more fields), the override's input block flows into both the LLM catalogue and runtime validation in lockstep.

**Authoring rule:** when a field is conditionally required (e.g. `offer_amount_egp` required only when `action: offer`), declare it `optional: true` and gap-collect it in pure YAML — either a `collect:` node (the engine-owned multi-field loop) or an `if`-guarded `render` + `pause` that re-runs reading `$state` (the validate → gap → goto-self pattern; the reply folds into `$state`). See [`publish-listing.flow.yaml`](../src/packs/real-estate/flows/tools/publish-listing.flow.yaml) (`collect:`) and [`ask-name.flow.yaml`](../src/packs/showcase/flows/tools/ask-name.flow.yaml) (single-field `if`/`pause`) for references.

**Collect free-text fields are auto-exposed in the tool's `inputSchema`.** A `collect` field with NO picker (`options:`/`prompt:`) is collected by the agent **re-calling the tool with the typed value**, so the platform automatically adds it to the tool's `inputSchema` as an optional string — otherwise Mastra's validator would strip the unknown key and the loop would re-ask forever. Picker fields stay unexposed (the agent must tap, not pass a raw id). Give such a field a `describe:` (the agent-facing schema description, distinct from the localized `label:`) when the name isn't self-evident; an explicit `input:` declaration always wins (richer type + describe).

## Reference packs

| Pack | Shape | What to copy |
|---|---|---|
| [`real-estate`](../src/packs/real-estate/) | Agent + flows + multi-channel domain | Manifest path-refs, pure-YAML flows (`collect:` / `dispatch:`), view-model rendering in YAML, channel hooks split into pure domain |
| [`showcase`](../src/packs/showcase/) | Capability demos (pure YAML) | One small flow per capability — interactive UI, gap/pause-resume, config/env reads, pre/post-turn hooks, media handler — all with zero pack TS |
| [`starter-kit`](../src/packs/starter-kit/) | Template / starter | Minimal pure-YAML pack: manifest + one flow (`assign` → `render` → `end`) + locale; the shape to copy when scaffolding a new pack |

## Pack HTTP routes — child process isolation

Pack admin routes (`src/packs/<id>/routes/index.ts`) run in a **sandboxed child process**, not in the main Fastify server. The platform:

1. Spawns one child process per pack via `PackChildProcess` (`src/platform/pack/child/bridge.ts`).
2. The child has **no platform credentials** — env is restricted to the pack's own `.env` vars + OS basics. No `SUPABASE_URL`, no `SUPABASE_SERVICE_KEY`.
3. Because the child can't construct IPC adapters itself (that would mean importing platform internals), the platform child entry ([`pack/child/entry.ts`](../src/platform/pack/child/entry.ts)) **builds** the IPC adapters and **injects** them via the `PackChildAdapters` contract (`{ db, getMessaging }`) on the route registrar's `ctx.childAdapters`.
4. DB access from routes uses the injected IPC `db` — the call chain (`db.from('leads').select(...)`) is serialised and sent to the main thread via IPC; the main thread replays it against the real schema-scoped client and posts the result back.
5. The main thread proxies HTTP to the child via `undici`, injecting `x-project-id`, `x-tenant-id`, `x-schema-name` headers; the child's `onRequest` hook reads them via `enterChildContext`.

**Wiring the injected adapters** — the pack's `routes/index.ts` wires `ctx.childAdapters` into its own zero-platform-import providers, then sub-routes use those:

```ts
// routes/index.ts
const registerRoutes = async (app, ctx, opts) => {
  if (ctx.childAdapters) {
    setDbProvider(() => ctx.childAdapters.db)              // → db/client.ts `db` proxy
    setChildMessaging(ctx.childAdapters.getMessaging)      // → routes/_child-messaging.ts
  }
  // …register sub-routes
}
```

Sub-route files then use the pack-local `db` proxy and `getIpcMessaging(channel)` — neither imports anything under `src/platform`. This inverted the old pack-side `child-adapters.ts` (deleted): construction is the platform's job, the pack only consumes injected handles.

URL shape: `/api/tenants/:tenantSlug/projects/:projectSlug/packs/<packId>/*`

## Minimal pack skeleton

```
src/packs/<your-pack>/
  manifest.yaml                # required — declares id, version, flows, agents, hooks
  pack.ts                      # optional — TS-typed `PackManifest` re-export if you prefer typed manifests
  register.ts                  # was required for the retired slim packs; absent for YAML-only auto-discovered packs
  db/
    migrations/                # SQL applied at provision time
    client.ts                  # `db` proxy delegating to platform adapter
  domain/                      # pure pack-domain code
  flows/
    <flow-id>.flow.yaml        # flow DSL (this + locale is all a pure-YAML flow needs)
    <flow-id>.locale.ar.yaml   # locale strings
    <flow-id>.primitives.ts    # OPTIONAL escape hatch — only when a flow needs TS the expr grammar can't express; default export { [name]: definePrimitive(...) }
  handlers/                    # channel-hooks.ts (pure domain functions; path-refs from manifest)
  prompts/                     # agent instructions (referenced from manifest.agents[].instructions)
  repositories/                # domain repos using pack `db` client
  services/                    # external integrations (Supabase, vision, storage, etc.)
```

YAML-only discovery: drop a `manifest.yaml` in `src/packs/<id>/` and [`pack/bootstrap.ts`](../src/platform/pack/bootstrap.ts) (`autoBootstrapAllPacks`) picks it up — no `register.ts` needed — the only packs that hand-rolled `definePack({ handlers })` registration were the retired slim packs.

## Pack file surface — parity checklist

A pack is auto-discovered from `manifest.yaml`, but several **optional** root files unlock behavior the platform won't give you otherwise. The recurring new-pack failure is "real-estate ships file X; my pack forgot it, so feature X silently doesn't work." Ship the ones you need:

| File (pack root) | Required? | What it gives you | If absent |
|---|---|---|---|
| `manifest.yaml` | **yes** | id, version, agents, flows, hooks, migrations, seeders | pack isn't discovered |
| `commands.yaml` | optional | slash commands (`/clear` → `close_conversation`, `noop`) | **no slash commands** — `/clear` is treated as plain agent text |
| `locale.<lang>.yaml` | optional | pack-level (cross-flow) strings, namespaced by `pack_id` | `$t("<pack_id>.key")` won't resolve |
| `messages.<lang>.yaml` | optional | platform-locale overrides (channel errors, etc.) | platform defaults |
| `taps.yaml` | optional | author-side tap-id conventions | — |
| `cases.yaml` | optional | casework: queues + case types + dispositions (see [casework README](../src/platform/casework/README.md)) | no cases — `case_open` fails on unknown type |
| `journeys/*.journey.yaml` | optional | customer-journey funnels for `track:` / `track_event` | `track:` silently skips |
| `xrm.yaml` | optional | XRM records: entities + pipelines + segments (see [docs/25](25-xrm-records.md)) | no records — `record_*` fail on unknown entity |
| `worklist.yaml` | optional | worklist: queues + per-entity routing/SLA/actions over any worked entity | the entity isn't worked (no queue, no operator actions) |
| `automation.yaml` | optional | standing sweep / campaign jobs upserted per project at provision | no standing jobs |
| `roles.yaml` | optional | **RBAC**: domain roles + their permission grants, assignable to users/teams (see [rbac README](../src/platform/rbac/README.md)) | only the platform-standard trio (`agent`/`supervisor`/`read_only`) is assignable |
| `db/migrations/*.sql` | if DB-backed | schema applied at provision (auto-discovered) | no schema |
| `db/seeders/*.sql` + manifest `seed_assets` | optional | reference/demo data on provision (see below) | empty tables |
| `flows/<id>.flow.yaml` (+ `.locale.<lang>.yaml`) | per flow | the flow + its strings | — |

Copy [`real-estate`](../src/packs/real-estate/) for the full set; [`starter-kit`](../src/packs/starter-kit/) ships the minimal stubs (`commands.yaml`, a namespaced `locale.ar.yaml`) so a scaffolded pack starts with the right shape.

## `roles.yaml` — declare WHO may do WHAT (RBAC)

Your pack knows what its records mean; the platform doesn't. So the pack declares the **domain roles** a
workspace can hand out, and the platform enforces them. A role is a key, a human `description:` (the console
renders it to the admin doing the assigning — write it for them, not for you), and **grants**:

```yaml
roles:
  regional_agent:
    description: >-
      Handles captain cases for the region queues bound to their assignment. Can approve
      refunds and unblock accounts, and reassign within reach.
    grants:
      record.case:                     # <domain>.<subject>
        view:       [queue, team]      # verb: scope (a list = union)
        note:       [queue, team]
        transition: [queue, team]
        act:        { scope: [queue, team], actions: [approve_refund, unblock] }
```

- **Verbs** are the fixed platform set — `view · create · update · note · transition · assign · act · archive`.
  A domain-specific operation is NOT a new verb: it's an **`act` action key** (your `worklist.yaml` actions,
  your `cases.yaml` dispositions, `run`/`send` on automation), which a role may allowlist.
- **Scopes** are `own | team | queue | all`. `queue` resolves against the queues bound **at assignment time**
  (or the role's default `queues:`), so ONE `regional_agent` serves every region — bind the Western team's
  assignment to `ops_western`, the Central team's to `ops_central`.
- **Resources** are `<domain>.<subject>`: `record.<your entity>` (incl. `record.case`), `automation.job`,
  `automation.campaign`, `journey`, plus wildcards (`record.*`, `*`). Everything is validated at boot against
  the securable-domain catalog — a typo'd entity, an undeclared action, or a scope the domain doesn't support
  is a boot error, not a silent hole.
- The platform merges a **standard trio** (`agent` / `supervisor` / `read_only`) under your roles, so you only
  author what the generic ones can't say. Declaring a role with one of those keys overrides it.
- **You cannot grant workspace-configuration power** (rotating a channel secret, editing flows) — that lives on
  the `TenantRole` axis, deliberately out of a pack's reach.

Reference: [`src/packs/kaiian/roles.yaml`](../src/packs/kaiian/roles.yaml) · full model + the enforcement audit
table: [`rbac/README.md`](../src/platform/rbac/README.md).

## Locale keys are NAMESPACED — bare `$t("key")` never resolves

Every `$t(...)` key MUST be `<namespace>.<key>`: a **pack-level** key (in `<pack>/locale.<lang>.yaml`) is namespaced by `pack_id` → `$t("clinic.field_picker_footer")`; a **flow-level** key (in `<pack>/flows/<flow>.locale.<lang>.yaml`) is namespaced by `flow_id` → `$t("book-appointment.preview.slot")`. A **bare** key with no `.` does NOT resolve — the runtime `t()` ([`flow-runtime/locale.ts`](../src/platform/flow-runtime/locale.ts)) returns the raw key string, so the literal key text leaks into the user's message. The flow linter's `locale-key-not-namespaced` rule (boot-time, error severity) fails the build on a bare key, so this is caught before it ships.

## Prompt variables — `{{namespace.key}}` (one engine, per-turn)

Agent prompt files (`instructions: - file: …`) and `working_memory.populate` may use
`{{namespace.key}}` placeholders, resolved by ONE engine (`renderMustache` + `buildPromptVars`,
[`manifest/placeholders.ts`](../src/platform/manifest/placeholders.ts)):
- **Variables:** `{{date.*}}` (`year`, `month_name_ar/en`, `weekday_ar/en`, `day`, `hour`,
  `minute_padded`, `next_month_name_ar/en`, `next_month_year`, `year_plus_1..10`, …),
  `{{pack.id}}`/`{{pack.version}}`, `{{channel.kind|from|display_name|lang}}`, `{{tenant.id}}`,
  and (working-memory only) `{{session.*}}`.
- **Fallbacks:** `{{channel.display_name || "there"}}` — used when the path is missing.
- **Unknown keys render verbatim** (so a typo like `{{date.yera}}` is visible, not silently empty).
- **Resolved PER TURN** — instructions are rendered inside the agent's dynamic `instructions`
  function, so `{{date.*}}` is the live turn time, never frozen at agent build. (`@mastra/core`
  ships no templating; this is our engine on its dynamic-instructions seam — `@mastra/editor`'s
  `renderTemplate` is intentionally not used.)
- **Date-awareness is opt-in:** include `{{date.*}}` in your own prompt (copy
  [`real-estate/prompts/current-date.md`](../src/packs/real-estate/prompts/current-date.md) or
  [`clinic/prompts/current-date.md`](../src/packs/clinic/prompts/current-date.md)).

## Agent prompt — ground it to real tools; don't recite data

A capable model **confabulates** when the prompt leaves room: it invents menu entries, claims capabilities the pack doesn't have, and recites plausible-but-fake facts (addresses, prices, hours, availability). Two prompt-authoring rules keep it honest:

- **Enumerate only the tools that exist.** The identity/instructions prompt lists the real tools and when to call each. Don't gesture at a feature ("see our locations") that no tool backs — the agent will offer it and then have nothing to call.
- **Make it call a tool for facts, not narrate them.** For any data the user might ask for that lives behind a tool, instruct the agent to *call the tool* and read its result — explicitly "don't invent or recite `<X>`." Reserve the prompt for behaviour/intent; let tool envelopes carry the data.

Corollary: if the prompt (or a tool description) advertises a capability, **build the flow that backs it** — the prompt and the tool surface must not promise more than the pack delivers.

## Shared DB extensions — declare them, never `CREATE EXTENSION` in a migration

A DB-backed pack's migrations run against a **per-install schema** (`search_path` = that schema, then `public`). A `CREATE EXTENSION` inside a migration installs the extension into whichever schema wins the race — Postgres puts an extension's functions in **one** schema — so every *other* install's `search_path` can't see it, and a function the extension provides then fails at runtime with "function … does not exist": install-dependent and hard to reproduce.

Declare the extensions the pack needs in the manifest instead:

```yaml
db_extensions: [pg_trgm, vector, "uuid-ossp"]
```

At provision the platform installs each one — or **relocates** an already-installed copy — into the shared `public` schema, which is on every install's `search_path`. Pack migrations then use the extension's functions freely and must **not** `CREATE EXTENSION` themselves. (On a managed/BYO store whose role can't relocate an extension, provisioning logs a notice and continues; the DB owner pre-installs it in `public`.)

## Seeders — reference/demo data on provision

A DB-backed pack can ship reference/demo data that loads when an install is provisioned with the **seed** flag (the console's "Seed demo data" checkbox, or a re-seed from the Database tab). It's **pure SQL** at provision time — any AI artifacts (embeddings, images) are precomputed at AUTHOR time and committed, so provisioning never calls a model.

- **`db/seeders/*.sql`** — listed (ordered) in `manifest.seeders:`. Run after migrations, tracked in a `_pack_seeders` ledger inside the install schema (idempotent). Make each seeder row-idempotent (`WHERE NOT EXISTS …` / `IF EXISTS … RETURN`) so a partial re-seed fills only gaps.
- **`manifest.seed_assets:`** — `{ file, table, column, match, array? }` entries map a committed JPEG in `db/seeders/assets/` to a DB column: the runner uploads it to the install's storage and writes the public URL into `<table>.<column>` for the matched row(s). (Storage is Supabase-backed, so this is a no-op on a bare-`postgres` store.)
- **`npm run seed:gen <pack>`** — the AUTHOR-TIME generator (`scripts/seed-gen.ts` → the pack's `db/seeders/_gen.ts`). With `OPENROUTER_API_KEY` set it (re)generates `*.generated.sql` (incl. `::vector` embedding literals) + the committed images; without a key it degrades (NULL embeddings, skipped images). Commit the output.
- **Seed every supported language.** A normalizer (script/orthography folding, case-folding, accent-stripping) only narrows variants *within* a language — it does **not** translate or transliterate *across* languages. If the pack matches/searches in more than one language, seed each searchable entity's names in **every** supported language; otherwise a query in one language never matches rows stored only in another.

See [`real-estate/db/seeders/`](../src/packs/real-estate/db/seeders/) (SQL + embeddings + compound images) and [`clinic/db/seeders/`](../src/packs/clinic/db/seeders/) (a `DO $$ … $$` demo block + doctor avatars), and the provisioning [`seeder-runner.ts`](../src/platform/provisioning/seeder-runner.ts) README.

## When in doubt

1. Check [`platform_sdk.ts`](../src/platform/platform_sdk.ts) (the `#sdk` alias) — is there an export for what you want?
2. Check the reference packs above — has someone done this already?
3. Check the memory entries in `~/.claude/projects/c--projects-ammar-2/memory/` — many cross-cutting "do this, not that" rules are documented.
4. Open a discussion before reaching into platform internals.
