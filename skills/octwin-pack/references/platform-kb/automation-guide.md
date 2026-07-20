# automation/

> 📘 **Module guide (business + authoring):** <https://claude.ai/code/artifact/67f9fc4a-8711-4b13-ab4e-8405339af436>
> — the automation engine training guide (operators: jobs / campaigns / what's observable; authors:
> declaring the lifecycle timeout + driving campaigns). Source:
> [`docs/artifacts/automation-module-guide.html`](../../../docs/artifacts/automation-module-guide.html)
> (rebuild + redeploy it when this module changes).

The generic **records-aware scheduled-sweep engine** — the third engine over XRM
records (after `worklist`'s human inbox): time/predicate-driven **rules** that act
on records, and **campaign** orchestration. It's the seam the four converging
`docs/BACKLOG.md` items named — casework SLA-escalation, worklist SLA,
notify-on-overdue-task, and campaigns-over-segments.

## Why this layer sits above worklist/casework

A sweep must call `compileWhere` / `listRecords` / `transitionStage` (xrm) and read
the worklist SLA overlay — all ABOVE `reminders` (rank 16) in the layer-DAG. So the
engine can't live in `reminders`; it's a new layer above `worklist`/`casework` that
composes them downward and reaches DOWN into `reminders` only for the opaque
per-recipient delivery it already does well. **`reminders` stays the low-level
delivery substrate; `automation` adds the orchestration.**

## Files

- **`contracts/job.ts`** — the `automation_jobs` row + per-kind config (Zod). Job
  kinds: `segment`, `stale_records`, `sla_sweep`, `task_overdue`.
- **`store.ts`** — CRUD + the ticker's `claimDue` (`FOR UPDATE SKIP LOCKED`) + `reArm`.
- **`services/sweep-service.ts`** — `runJob` dispatch. `segment` (notify each matched
  contact via reminders — deduped once per cooldown window — or transition),
  `stale_records` (delegate to xrm `runAutoTransition` — the E2 idle timeout), and
  `sla_sweep` (relay a reassurance message to the CUSTOMER of a record breached
  past its `sla_due_at` column, once per breach via `sla_notified_at`) are wired.
  `task_overdue` is recognized but **deferred** — it targets an OPERATOR (a task has
  no customer), which needs an operator channel (see BACKLOG); so does `sla_sweep`'s
  operator-reassign half.
- **`registry.ts` + `services/provision-jobs.ts`** — pack-declared standing jobs. A
  pack ships `<packRoot>/automation.yaml` (`jobs:` list); the loader registers it into
  `AutomationRegistry`, and `ensureJobsForProject` upserts one `automation_jobs` row
  per `key` at provision (idempotent), via the injected `applyAutomationJobs` hook
  (`src/automation-jobs-provision.ts` — provisioning is below automation in the DAG).
  The shipped example is `src/packs/ecommerce/automation.yaml` (abandoned-cart recovery).
- **`services/campaign-service.ts`** — `createCampaign` (a `campaign` xrm record at
  `draft`) + `sendCampaign` (resolve the audience → fan out one `reminders` send per
  matched contact, tagged `source_campaign_id`; advance the stage). Reuses reminders
  for delivery/retry/ledger + a single-query progress rollup — no recipient table.
- **`executor.ts`** — the MAIN ticker (`startAutomation`, gated by
  `AUTOMATION_SCHEDULER_ENABLED`); one pass claims due jobs, runs each, re-arms
  `next_run_at` on the claim txn.
- **`boundaries.ts`** — the only seam other layers import.
- Migration **`db/migrations/070_automation_jobs.sql`** — `automation_jobs` +
  `scheduled_messages.source_campaign_id`.

## Business

The flagship use case is **abandoned-cart recovery**: a `segment` job over `cart`
records that are `open` and idle nudges each shopper ("you left items in your cart"),
and a `stale_records` job runs the cart's declared `auto_transition` (`open →
abandoned` after 48h idle). Both need zero pack code — they read the `cart` system
entity + its declared segment/timeout. The same engine powers kaiian SLA-escalation
+ `stalled_applications` re-engagement and clinic booking nudges once their
`sla_sweep` / operator paths land.

## Console

Operators manage jobs + campaigns at **Automation** (project sidebar) — pause /
resume / run-now a job (with its `last_result`), and view campaigns with live send
progress + a "Send" for a draft. Backed by `src/routes/automation.ts`. The ticker is
off by default (`AUTOMATION_SCHEDULER_ENABLED`).

## When to add files here

A new job KIND → a `runX` branch in `sweep-service.ts` + a config shape in
`contracts/job.ts`. A new delivery policy stays in `reminders`. Operator-targeted
actions (reassign, operator reminders / `task_overdue`) unlock when per-operator
identity/channel lands (see `docs/BACKLOG.md`).
