# `casework` — platform case-management module

Generic, business-agnostic **case / ticket management**. Turns a conversation
into a durable, human-worked **case** that outlives a single turn: it is opened
by a flow, routed into a **queue**, worked by a 2nd-line human in the console,
and resolved — with the AI agent as the **sole voice to the customer**
(mediated model). A **pack declares its case work in `worklist.yaml`** (`work.case`)
and the platform stays domain-blind (invariant #6).

> **Casework is a thin PRESET over the worklist engine — it has NO grammar of its
> own.** A case is the [`case` XRM system entity](../xrm/system/entities.ts)
> (fields/stage/timeline/number) worked via the generic
> [`worklist`](../worklist/README.md) engine — queues, routing, SLA, and the
> decision **action** library all declared in the pack's `worklist.yaml`
> `work.case` block. Casework contributes only the case-DOMAIN vocabulary that
> doesn't generalize: the **standard decision set** (`resolve`/`reject`/
> `request_more_info`/`escalate` — [`preset.ts`](preset.ts), merged into
> `work.case` at registration by `applyCaseworkPreset`), the mediated-relay copy
> (`composeDecisionMessage`), and the strict locale-coverage gate. The pack's
> old cases.yaml grammar / registry / default workflow / standard-disposition set
> are gone (deleted) — the case pipeline is the entity's (`CASE_STATUS_TRANSITIONS`), the
> ONE workflow. `case-service` is a contract-preserving adapter (the `CaseRow`/
> timeline shapes are unchanged, so the `case_*` primitives + console are untouched).

> Module guide Artifact: <https://claude.ai/code/artifact/6f37511b-4dc0-4b61-8802-062f74cfd034>
> (source [`docs/artifacts/casework-module-guide.html`](../../../docs/artifacts/casework-module-guide.html)).
> First consumer: the Kaiian/Waslini ride-hailing support pack.
>
> Siblings: [`xrm`](../xrm/README.md) (the record engine a case now lives on) +
> [`worklist`](../worklist/README.md) (the generic queue/routing/SLA/actions engine
> a case is worked through). The cases-as-xrm-entity convergence — once a backlog
> evaluation — shipped: `case` is a system entity, casework the preset over it.

## For operators (business)

- A **case** is a complaint / dispute / escalation with a lifecycle
  (`open → in_progress → awaiting_customer → resolved → closed`, + `reopened`),
  a **queue** (e.g. a regional team), structured **fields**, a **priority/SLA**,
  and an append-only **timeline**.
- **Mediated model (v1):** the 2nd-line human never messages the customer. They
  pick a **disposition** (a declared decision like `approve_refund`,
  `request_more_info`, `resolve`, `reject`, `escalate`) + an optional **internal
  note**. `recordDecision` records the decision (the internal note stays
  operator-only) and relays the update to the customer through the platform
  **`notify`** path (casework → notify → the customer's channel + thread) — no
  templating, the change is relayed as-is; `notify`'s pre-turn drain injects a
  `memory_note` so the agent has context when the customer next replies. The
  agent composes free-text only on the customer's own turn.
- **Every relay leaves a timeline record.** `recordDecision` writes a
  customer-visibility `outbound_msg` event per relay attempt — payload
  `{ body, action, notified, failure_reason? }` — **even when the notify
  failed**, so the console timeline shows the failed delivery (a red
  "delivery failed" badge) instead of silently claiming the customer heard
  back. `notified` reflects the real `NotifyResult` (a returned
  `state:'failed'` counts as not-notified, not just a throw). The relayed copy
  carries the bare `#<case_number>` ticket ref (the pack's display prefix stays
  pack-side).
- **Queues = teams.** A case routes to a queue by a static `queue` or by a field
  (`route_by: region` → `ops_<region>`, or an explicit `map:`). A `route_by`
  type may declare a **`fallback:`** queue — the safety net when the field is
  missing or (with a `map:`) unmapped. With a declared `map`, an unmapped value
  no longer falls through to the `ops_<value>` convention (that lazily minted
  phantom queue keys). No fallback → the case opens **unrouted** (`queue_key
  NULL`, warn-logged); the console "All queues" inbox includes unrouted cases and
  offers an "Unrouted" filter so they can't silently strand. Queues are a
  **keys-only registry concept** (declared in `worklist.yaml`) — there is no queue
  table and no queue UUID; RBAC and the record's `queue_key` column bind to the key.
- **Access is RBAC (live).** Who may see and act on which queue is decided by the
  caller's pack-declared **role grants** (`view`/`note`/`transition`/`assign`/`act`
  on `record.case`, scoped `own`/`team`/`queue`/`all`), resolved per request from
  their `Principal`. Platform operators + tenant `owner`/`admin` see everything; a
  `member` sees only what their assigned roles grant; a `viewer` reads but cannot
  write. A case with no queue is reachable only under an `all`-scoped grant.
  `access-service.ts` is now a thin derivation over the engine — see
  [`rbac/README.md`](../rbac/README.md).
  *(The inert `worklist_members` model this once anticipated was retired in migration 077.)*

## For pack authors (technical)

Declare case work in `src/packs/<id>/worklist.yaml` under `work.case` — a case
type is a one-liner; the standard decision set (`resolve`/`reject`/…) is merged in
automatically by the casework preset (declare only CUSTOM actions):

```yaml
queues: [customer_ops, ops_central, ops_western, ops_eastern, ops_northern, ops_southern]
work:
  case:
    type_field: case_type
    actions:                          # OPTIONAL custom decisions beyond the standard set
      approve_refund: { params: [{ name: amount, type: number }], relay: true }
    types:
      extra_charge:      { queue: customer_ops, priority: high, actions: [approve_refund] }
      general_complaint: { queue: customer_ops }     # one line
      unpaid_fare:       { route_by: region }         # region → ops_<region>
```

Then drive cases from flows via the `case_*` primitives:

- `case_open { type, fields }` — resolves the queue + priority + SLA from the
  `worklist.yaml` `work.case` config and stores `fields` verbatim. → `{ case_id,
  case_number, sla_hours, sla_due_at }`. `case_number` is a **per-project** monotonic
  ticket ref (the record's `record_number`, allocated via `xrm_counters`); the pack
  composes the customer-facing reference (e.g. `KAI-<case_number>`) and uses
  `sla_hours` to set the response expectation.
- `case_list { contact_id?, status? }` — the contact's cases (powers the
  "status of my complaint?" tool natively; no external lookup).
- `case_get { case_id }` — a case + its timeline.
- `case_transition { case_id, to_status, note? }` — workflow-validated status move.
- `case_attach { case_id, media_id, caption? }` — attach an uploaded `MEDIA-` ref to a
  case (evidence). The primitive (rank 24) orchestrates both sides of the link: media's
  `linkMediaToCase` (queryable `media_assets.case_id`) + this module's `appendAttachment`
  (an `attachment` timeline event) — casework never imports media (layer-DAG). The console
  case-detail **Attachments** card resolves refs → signed URLs at the route tier. See
  `docs/19-platform-media.md` §8b.

The `collect:` flow owns field collection; the case type's optional `collect:`
list is only an inbox display-order hint, never a re-typed schema.

### The casework locale contract (boot-enforced)

A pack that works the `case` entity must ship a pack-level `locale.<lang>.yaml`
covering, per language: `cases.<type>` (one label per `work.case.types` id),
`status.<state>` (the `case` entity's pipeline stages), and `decision.<action>`
(standard preset ∪ pack customs). `$enum_label` resolves labels through these
keys and falls back **silently** otherwise — so coverage is validated at
boot/test-load in **both directions** (missing keys AND orphan keys naming
undeclared values fail) by `validateCaseworkLocaleCoverage`
([`locale-coverage.ts`](locale-coverage.ts)), wired from `definePack` step 5i
(over the resolved worklist vocabulary). A companion flow-lint rule
(`picker-enum-label-missing` / `picker-case-type-unknown` /
`picker-input-enum-mismatch` in `flow-runtime/lint`) keeps static pickers,
`input:` enums, and the declared case types in lockstep.

**The exact values to cover** (so `status.*` / `decision.*` are complete and no boot check fails) —
copy these into your pack `locale.<lang>.yaml`:

- **Status** — the `case` pipeline stages, and the only legal `case_transition` `to_status` targets
  (`CASE_STATUS_TRANSITIONS`, [`../xrm/system/entities.ts`](../xrm/system/entities.ts)):
  `open` · `in_progress` · `awaiting_customer` · `resolved` · `closed` · `reopened`.
- **Priority** — the `case.priority` select: `low` · `normal` · `high` · `urgent`.
- **Standard decisions** — merged into every case type (add customs under `work.case.types.<t>.actions`),
  each with the stage it moves the case to ([`preset.ts`](preset.ts) `CASE_STANDARD_ACTIONS`):
  `resolve` → `resolved` · `reject` → `resolved` · `request_more_info` → `awaiting_customer` · `escalate` → `in_progress`.

So a pack needs `cases.<type>` (per declared type), `status.open … status.reopened` (all six), and
`decision.resolve … decision.escalate` (+ any custom actions) in each language.

Flow YAML can read the declared type ids without re-typing them via the
runtime-bound **`$work_types(entity, route_by?)`** function (e.g. kaiian's region
gate: `$in($state.issue_type, $work_types("case", "region"))`).

## Layout

| File | Role |
|------|------|
| `contracts/case.ts` | Stored row + timeline value types (the `CaseRow`/`CaseEventRow` projection contract). |
| `preset.ts` | `CASE_ENTITY` + `CASE_STANDARD_ACTIONS` (the standard decision set) + `applyCaseworkPreset` (merges them into a pack's `work.case` at registration; pack wins by key). |
| `locale-coverage.ts` | `validateCaseworkLocaleCoverage` — the boot-time locale contract (pure; `define.ts` resolves the vocabulary + bridges the locale map in). |
| `services/case-service.ts` | Contract-preserving adapter over `#platform/xrm` (`case` record + timeline via `saveRecord`/`transitionStage`/`appendRecordEvent`) + `#platform/worklist` (queue/SLA via `enrollAt`; decisions via `applyRecordAction`); reads the disposition catalog + queue names from `WorklistRegistry`, transition legality from the `case` entity pipeline. Projects back `CaseRow`/`CaseEventRow` (queue/assignee/SLA read straight off the record; the terminal-close timestamp is the `completed_at` column). |
| `services/access-service.ts` | The single authorization seam — a thin derivation over the [`rbac`](../rbac/README.md) engine (`assertCanForRecord` / `visibleQueueKeys` on `record.case`). |
| `boundaries.ts` | The only public import surface. |

Storage: NONE of its own — a case is the `case` XRM system entity (`xrm_records`,
migration 039) whose generic work-management columns (`queue_key`/
`assignee_principal`/`sla_due_at`, migration 082) carry queue/SLA. The former
`037_casework.sql`/`038_case_reference.sql` tables are retired, and the
`worklist_items`/`worklist_queues` overlay was dropped (migration 083). No loader
of its own — a case's work config rides the `worklist.yaml` loader; the casework
preset + locale gate run at `definePack` step 5i.
Primitives: `src/platform/primitives/casework/*` (unchanged; delegate to the adapter).

## Invariants

- **Platform stays domain-blind.** No pack id / domain term here; case types are
  pack-authored data in the registry, instances carry `pack_id` as plain text.
- **The platform feeds the thread, not the pack.** A decision relays to the
  customer via the platform `notify` path + its pre-turn drain (the same
  mechanism as real-estate's buyer↔seller relay) — casework never synthesises an
  inbound or forces an agent turn.
- **Access decisions live only in `access-service.ts`**, which now delegates to the
  `rbac` engine (`assertCanForRecord` / `visibleQueueKeys`). Casework itself holds no
  authorization logic — one seam, one place to audit.
