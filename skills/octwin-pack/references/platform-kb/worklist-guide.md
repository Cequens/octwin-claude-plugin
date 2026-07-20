# `worklist` — generic human-inbox overlay over XRM records

Turns **any** XRM entity into a queue-routed, human-worked inbox item — a `case`,
an `application` needing approval, an `order` needing review. It is a **storage-free
XRM-composing adapter**: the four per-record work facts (`queue_key`,
`assignee_principal`, `sla_due_at`, `sla_notified_at`) are generic columns ON the
record itself (`xrm_records`, migration 082), so `worklist` owns the ROUTING policy
+ inbox shaping, not a table. The record — fields, stage, timeline, number, contact —
stays pure XRM. `worklist` **writes no system entity of its own**; that's what keeps
it generic. (Queues are a **keys-only registry concept** from `worklist.yaml` — no
queue table, no queue UUID; RBAC binds to the key.)

> Generalizes what `casework` used to own (queues, routing, SLA, dispositions,
> access). `casework` is now a thin preset that enrolls a `case` entity here.

## For operators (business)

- A **queue** is a team a record routes to. The **work-inbox** lists a queue's open
  items (record #, title, contact, stage, SLA) across any worked entity; unrouted
  items surface in an "Unrouted" filter so nothing strands.
- Routing is declarative: a static queue, or `route_by` a field (`region` →
  `ops_<region>`), with an optional explicit `map:` and a `fallback:` safety net.
- **Record actions** (operator decisions that transition the record + relay to the
  customer), **SLA** deadlines, and per-queue **access** are all live — a decision
  is an `actions:` entry (`applyRecordAction`), SLA is `sla.resolve_within_hours`,
  and access is a `queue`-scoped RBAC grant. The `casework` preset builds the `case`
  entity's standard decision set on top of this.

## For pack authors (technical)

Declare one compact `src/packs/<id>/worklist.yaml` — queues + which entities are
worked + how they route:

```yaml
queues: [customer_ops, ops_central, ops_western]
work:
  case:                       # an XRM entity that becomes worked
    type_field: case_type     # sub-type discriminator (enables per-type routing)
    route_by:   region        # default: region → ops_<region>
    fallback:   customer_ops
    types:                    # per-type overrides (keyed by case_type value)
      extra_charge: { queue: customer_ops }
```

Then drive it from flows: **`work_open { entity, type?, fields, contact_id? }`** —
saves the XRM record AND enrolls it (routed to its queue). Reads/stage-moves are
plain XRM (`record_get`/`record_list`/`record_stage`); the inbox is a console
surface over `listItems`.

## Layout

| File | Role |
|------|------|
| `contracts/spec.ts` | `worklistFileSchema` — queues + per-entity routing. |
| `resolver.ts` | `validateWorklistFile` + `buildPackWorklist` + `resolveQueueKey`. |
| `registry.ts` | `WorklistRegistry` — in-process per-pack lookup. |
| `services/worklist-service.ts` | `enroll`/`enrollAt` (set `queue_key` + `sla_due_at`), `reassign`, `assign`, `listItems`/`countItems`/`getItem`, the SLA sweep — all composing the XRM record columns. |
| `boundaries.ts` | The only public import surface. |

Storage: NONE of its own — the work facts are columns on `xrm.xrm_records`
(migration 082); the former `worklist_queues`/`worklist_items` overlay tables were
dropped (migration 083). Loader: `manifest/worklist.ts`, wired into `definePack`
step 5i. Primitive: `primitives/worklist/work_open`.

## Invariants

- **Platform stays domain-blind.** No pack id / ticket vocabulary here; queues +
  routing are generic work-management. The case-domain preset (standard dispositions,
  the support workflow) lives in `casework`.
- **The worked object is an XRM record.** `worklist` never re-stores it; it adorns
  it via the record's own work columns (`queue_key`/`assignee_principal`/`sla_*`)
  and reads XRM for display — the `worklist_items` overlay table was dropped (083).
- **Access is RBAC; assignment is worklist's.** `assign` writes `assignee_principal`
  (a `user:`/`team:` ref) and is gated by the `assign` verb. **Who may work which queue
  is no longer a worklist concern**: the bare `worklist_members` grant (and its
  `addQueueMember` writer) was retired and dropped in migration 077 — queue access is now a
  `queue`-scoped grant on a pack-declared role, bound to queues at assignment time
  (see [`rbac/README.md`](../rbac/README.md)). `listAccessibleQueueKeys` is a thin
  derivation over the caller's resolved grants; enforcement is threaded from
  `request.principal` route-side (`resolveRecordAccess`). The assignee is notified via
  `#platform/ops-inbox`.
