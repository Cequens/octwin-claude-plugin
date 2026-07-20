# journey/

> 📘 **Module guide (business + authoring):** <https://claude.ai/code/artifact/54836ea5-d38c-40d4-a7c8-b5902bee5f3e>
> — start here for the concept, reading the console, and declaring/tracking a journey. Source:
> [`docs/artifacts/journey-module-guide.html`](../../../docs/artifacts/journey-module-guide.html) (rebuild + redeploy it when this module changes).

Generic, domain-agnostic **customer-journey analytics** — a thin **declaration +
adapter layer over the XRM record engine**. Packs declare one or more journeys
(each: **stages → goals → events**) in
`src/packs/<id>/journeys/<journey_id>.journey.yaml`; flows emit progress with the
`track:` DSL node (or the `track_event` primitive); the platform turns that
telemetry into funnels, goal-completion, trends/cohorts and cost/attribution. The
journey is 100% pack-authored — **no pack id or domain namespace appears in this
layer** (platform-agnostic invariant).

## A journey run IS an XRM record

Journey has **no tables of its own** (the casework/surveys pattern). Each declared
journey compiles to a **synthesized `journey_<id>` XRM entity** (`compiler.ts`),
and a journey **run** is an ordinary `xrm_records` row: the stage is the record's
pipeline stage, goals are **milestones** (`xrm_records.milestones` /
`milestone_value_sum`), re-entry is the entity's **`enrollment:`** policy, and the
event stream is `xrm_events`. This means every generic XRM surface — the records
browser, segments, automation sweeps, campaigns — can operate on journey runs, and
the funnel analytics are the generic `xrm/services/funnel-service.ts`. (See the
mapping in [`docs/24-journey-lifecycle-benchmarks.md`](../../../docs/24-journey-lifecycle-benchmarks.md).)

## Why this layer sits directly above `xrm`

The adapter + analytics wrappers value-import `#platform/xrm/boundaries.js` (the
record engine); the compiler imports the xrm spec schema. A layer may import only
layers at or below its rank, so `journey` sits **just above `xrm`**. It is itself
value-imported by `flow-runtime` (the `track:` handler), `primitives` (the
`track_event` primitive), `pack` (boot: compile + register), and the
journey-analytics routes (reporting) — all higher layers. It imports `xrm`, `kernel`
(tenant + pack context), `utils`, and `db`; it must **never** import
`conversation`/`manifest`/`agent`/`pack` — the optional conversation-timeline
mirror (`subtype: 'journey_event'`) is done at the high-layer call sites, not here.

## Files

- **`spec.ts`** — Zod `journeySpecSchema` + `JourneySpec`/`ResolvedJourney` types
  (stages with `order`, goals with optional `stage`/`value`, events with optional
  `advances_to`/`completes`, `lifecycle:`). Ids are `^[a-z][a-z0-9_]*$` — they
  become XRM entity/stage/milestone keys.
- **`resolver.ts`** — `validateJourneySpec` (cross-field integrity Zod can't
  express) + the per-pack lookup indexes + the **cross-journey index** used to
  infer which journey a bare `track:` id belongs to (and to detect ambiguity) +
  `planJourneyWrites` (fan one authored event out into its derived
  stage_enter / goal_reached writes).
- **`compiler.ts`** — `journeyToEntitySpec` compiles a journey into its
  synthesized `journey_<id>` XRM entity (pipeline ← stages, `milestones:` ←
  goals, `enrollment:` ← lifecycle); `journeyEntityKey` / `journeyIdFromEntityKey`
  address it. Consumed by `pack/define.ts` (boot merge) + the adapter.
- **`registry.ts`** — `JourneyRegistry`: resolved journeys per pack, populated at
  boot; read by the `track:` handler/primitive (write planning) + the
  journey-analytics routes (labels + declared stage order for reporting). Mirrors `TapRegistry`.
- **`store.ts`** — the **XRM adapter**: `recordJourneyEmission` (the single write
  entry — enrollment → stage transitions → milestones → a `signal` event, wrapped
  in a never-throw try/catch so telemetry can't fail a turn) + two contact readers
  over the record engine: `readContactJourneyState` (per-turn `$journey.*` source,
  project-scoped via the ALS) and `listJourneyStatesForContact` (tenant-fenced read
  of the contact's `journey_*` records across ALL of a tenant's projects — the
  contact-detail "Journeys" card; label enrichment happens at the route tier
  because journey sits below `pack-meta`).
- **`stores/analytics.ts`** — thin, contract-preserving **wrappers** over
  `xrm/services/funnel-service.ts`: resolve the synthesized entity, map the journey
  filter (`runs` → `records`), and re-shape the generic output back into the journey
  response types (`stage_id`/`goal_id`, declared `order` as rank, stage/goal labels).
  `listStageRuns` is the funnel drill-down (runs currently at a stage — the records'
  live `xrm_records.stage`).
- **`types.ts`** — the small value shapes (`JourneyEventKind` for planned writes,
  `JourneyContactStateRow` for the `$journey` binding).
- **`boundaries.ts`** — the only seam other layers import.

## Lifecycle (re-entry / progression) → XRM enrollment + pipeline

Each journey declares an optional `lifecycle:` policy (industry-standard names —
benchmarks + the full mapping in [`docs/24-journey-lifecycle-benchmarks.md`](../../../docs/24-journey-lifecycle-benchmarks.md)),
all defaulted to lifetime / forward-only / single-run. The compiler maps it onto
the generic XRM engine:

- `reentry: none | after_exit | anytime` + `cooldown_days` + `idle_reset_days` →
  the entity's `enrollment:` policy (`after_exit` → `after_completion`).
  `resolveEnrollment` decides at write time whether an emission reuses the
  contact's latest run or mints a fresh record (a re-purchase is a second run —
  Amplitude "hold properties constant").
- `progression: monotonic | free` → the pipeline's `progression` (monotonic =
  forward-only; a backward `track:` is a swallowed no-op in the adapter).
- `terminal` (goal/stage) → the pipeline's terminal stage; the lifecycle-terminal
  goal is a `terminal: true` milestone, so reaching it completes the run.
- Reporting supports a **conversion window** (`conversion_window_days`).

## Business

Operators get a per-project, per-journey analytics surface: where customers drop
off, how many convert and how fast, whether conversion is improving, and which
flows/agents drive conversions vs. cost. Because a run is now a record, those same
funnels are **actionable** — a "stuck 7 days in Consideration" segment can be swept
by automation or worked from the inbox. Funnel
ordering always comes from the pack's declared stage `order` — never a platform
constant.
