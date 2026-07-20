# 27 — Scheduling (slot/booking engine over XRM resource records)

> Module guide Artifact (operator + author training): <https://claude.ai/code/artifact/ae3cf673-086b-4bf5-a0b0-6247619d67e3>
> — source [`docs/artifacts/scheduling-module-guide.html`](artifacts/scheduling-module-guide.html).

The **scheduling module** ([`src/platform/scheduling/`](../src/platform/scheduling/),
[README](../src/platform/scheduling/README.md)) gives every pack a generic,
domain-blind **appointment/reservation** capability: any XRM entity becomes a
**bookable resource** (a doctor, room, table — a plain XRM reference record made
findable by XRM search) with recurring **availability**, date **exceptions**, and
a **race-free capacity ledger**; each **booking is the `booking` XRM system
entity**. No pack DB, no bespoke SQL.

Design lineage: the same `commerce`→`orders`→`xrm` split
([`docs/23`](23-commerce.md), [`orders/README`](../src/platform/orders/README.md)) —
the module owns the engine data (availability + slots), the transactional artifact
is an XRM record. It generalizes the clinic pack's former bespoke
`doctor_sessions`/`session_exceptions`/`available_slots`/`book_appointment` SQL.

## 1. Where it sits

- **Layer**: `scheduling` sits after `orders`, before `flow-runtime`
  ([`scripts/layers.mjs`](../scripts/layers.mjs)) — it WRITES XRM records
  (`saveRecord`) so it must outrank `xrm`. Imports `db`/`kernel`/`utils`/`xrm`/
  `pack-meta`; **not** `notify`/`reminders` (reminders are composed flow-side via
  `schedule_notify`). Public seam:
  [`boundaries.ts`](../src/platform/scheduling/boundaries.ts).
- **Schema**: migration [`060_scheduling.sql`](../src/platform/db/migrations/060_scheduling.sql)
  — `scheduling_availability_rules`, `scheduling_availability_exceptions`,
  `scheduling_slots` (the atomic capacity ledger), all keyed to an
  `xrm_records.id` (the resource); plus the `scheduling_available_slots` SRF +
  `scheduling_slot_capacity`. Control-plane platform DB, app-layer `project_id`-scoped.
- **Definitions in-process**: `SchedulingRegistry` populated at boot by `definePack`
  step 5g from `scheduling.yaml` (loader [`manifest/scheduling.ts`](../src/platform/manifest/scheduling.ts)).

## 2. Authoring: `scheduling.yaml`

Names which XRM entities are bookable + their slot defaults + reminder offsets.
See [`src/platform/scheduling/README.md`](../src/platform/scheduling/README.md)
§"For pack authors" for the full grammar; the shipped example is
[`src/packs/clinic/scheduling.yaml`](../src/packs/clinic/scheduling.yaml)
(`doctor` resource, T-24h/T-2h reminders).

## 3. Slot engine + atomic reservation

The `scheduling_available_slots` SRF expands recurring rules across a date window,
drops `closed` days, adds `extra` one-offs, subtracts the ledger's `booked`, and
returns **structured** rows only (`slot_start`/`slot_end`/`slot_minutes`/
`remaining`/`dow`/`local_date`/`local_time`) — **no locale labels in SQL**
(invariant #6); the pack composes the label via `$t`/`$enum_label`. Reservation
claims a seat with a single conditional upsert (`ON CONFLICT … WHERE booked <
capacity`), fixing the count()-then-INSERT TOCTOU race, then writes the booking
record; a record-write failure compensates the seat (two-store composition, same
caveat as `cart_submit` — a `db.withTransaction` seam is backlogged).

## 4. Primitives · reminders · console

- **Primitives** ([`src/platform/primitives/scheduling/`](../src/platform/primitives/scheduling/)):
  `slot_list`, `booking_reserve`, `booking_cancel`, `booking_reschedule`. Full port
  contracts in [`16b-platform-stdlib-primitives.md`](16b-platform-stdlib-primitives.md).
  Booking **reads/status moves reuse the XRM primitives** (`record_get`/`record_list`/
  `record_stage`/`$xrm.booking`); resource **search** is `record_search`.
- **Reminders**: `booking_reserve`/`_reschedule` return each declared offset's
  absolute `send_at`; the flow passes them to the existing `schedule_notify`
  (revokes via `cancel_scheduled`). The module never imports notify.
- **Booking pipeline**: the `booking` XRM system entity's `BOOKING_STATUS_TRANSITIONS`
  ([`xrm/system/entities.ts`](../src/platform/xrm/system/entities.ts)) —
  `requested → confirmed → checked_in → completed / cancelled / rescheduled / no_show`.

## 5. Deferred (see [`docs/BACKLOG.md`](BACKLOG.md))

`db.withTransaction` for the reserve two-store composition · soft holds with TTL
(reserve a seat during a multi-turn collect) · console availability editor
(scheduling nav cluster) · waitlists / overbooking policy / inter-slot buffers ·
a `get_scheduling_analytics` tool · pack `demo:` seeding of resources + availability.
