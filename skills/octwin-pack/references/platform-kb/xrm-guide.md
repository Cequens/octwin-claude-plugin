# 25 — XRM records (pack-declared custom objects)

> 📘 **Business + authoring guide (Artifact):** <https://claude.ai/code/artifact/1d18a4e9-2e25-4c20-91ff-fa11376ec335>
> (source: [`artifacts/xrm-module-guide.html`](artifacts/xrm-module-guide.html)). This doc is the
> in-repo detail behind the guide.

The **xrm module** gives every pack a declarative, domain-blind record system:
one compact `src/packs/<id>/xrm.yaml` declares **entity types** (typed fields +
optional stage **pipeline** + to-many **relations**) and saved **segments** —
and the platform provides durable storage, per-entity human numbering, an
append-only timeline, follow-up **tasks**, the `record_*` / `task_*` flow
primitives, the console **Customer OS** surface, the `$xrm` context binding,
and `get_xrm_analytics`. No migration, no TS, no repository.

Design lineage: the metadata-driven object model is borrowed from Twenty CRM,
translated into this platform's casework shape (fields JSONB + in-process
registry + explicit-projectId services).

## 1. Where it sits (architecture)

- **Layer**: `xrm` sits in the casework band (`scripts/layers.mjs` — after
  `commerce`/`casework`, before `flow-runtime`). Imports `db` / `kernel` /
  `utils` / `commerce` (Money). Public seam:
  [`src/platform/xrm/boundaries.ts`](../src/platform/xrm/boundaries.ts).
- **Schema**: migration
  [`039_xrm.sql`](../src/platform/db/migrations/039_xrm.sql) — `xrm_records`
  (instances; `fields JSONB`, **no domain columns**, per-project per-entity
  `record_number`, first-class `contact_id`, `stage`, soft `archived_at`),
  `xrm_counters`, `xrm_links` (to-many edges), `xrm_events` (append-only
  timeline — the canonical feed a future external-CRM sync adapter consumes),
  `xrm_tasks`. All in the `xrm` schema of the application DB (migration 080 carved
  them out of `public`; the platform pool runs `search_path=public,xrm`); every
  query is app-layer-scoped by `project_id`.
- **Definitions live in-process**: `XrmRegistry` is populated at boot by
  `definePack` step 5f from `xrm.yaml` (loader:
  [`manifest/xrm.ts`](../src/platform/manifest/xrm.ts)). The DB stores only
  instances — exactly the casework/journey pattern.
- **Resolution artifacts** (built once at registry-build): per-entity
  **field validators** (`validators.ts` — create/update Zod pairs; money
  decimals normalise to the commerce `{ value, offset }` shape), **compiled
  segment filters** (`filters.ts` — see §4), and **resolved pipelines**
  (`pipeline.ts`).
- **Activity stamp**: every service write (fields / stage / note / link /
  task-on-record) bumps `xrm_records.updated_at`, so `updated_within_days`
  recency filters double as activity/staleness filters — no extra column.

### Relation to the sibling modules

| Module | Split |
|---|---|
| **casework** | Tickets vs pipelines. Casework = an inbound *problem to resolve* (queue-routed, disposition-closed, customer-mediated). XRM = a *relationship progressing through stages* (a deal, an application). The kaiian pack ships BOTH side-by-side. Contracts deliberately rhyme — "cases as an XRM system entity" is a backlogged evaluation, not a rewrite. |
| **journey** | Journey is a thin declaration + adapter layer that sits ON this engine: each declared journey compiles to a synthesized `journey_<id>` entity and a run IS an `xrm_records` row (stage = pipeline stage, goals = milestones, re-entry = `enrollment:`). Its funnel analytics are the generic `funnel-service.ts`. (Formerly journey kept its own append-only tables + a state trigger, deliberately uncoupled from XRM — retired in the journey→XRM convergence.) |
| **contacts** | Records LINK to the platform `contacts` table (first-class `contact_id`, `contact`-typed fields, contact relations); xrm never duplicates person identity. |
| **notify / reminders** | Delivery infrastructure xrm *consumes*. Customer follow-ups go through `schedule_notify` from flows. There is deliberately NO automatic task-reminder path in v1 (task reminders target operators, and operator identity doesn't exist yet — see `docs/BACKLOG.md`). |

## 2. Authoring: `xrm.yaml`

One file per pack. Convention-first — the minimal entity is two lines:

```yaml
entities:
  note_topic: { fields: { name: text } }
```

**System entities.** The platform additionally ships standard entities in the
same grammar — declared in
[`xrm/system/entities.ts`](../src/platform/xrm/system/entities.ts) and available
to **every** pack through the registry fallback. The shipped set:
`order` (strict `ORDER_STATUS_TRANSITIONS` pipeline — [`orders`](../src/platform/orders/README.md)),
`product` (the WhatsApp/Meta commerce catalog item; `dedupe_by: retailer_id`, `search: { text, semantic }` —
[`catalog`](../src/platform/catalog/README.md)),
`booking` (`BOOKING_STATUS_TRANSITIONS` — [`scheduling`](../src/platform/scheduling/README.md)),
`survey_response` ([`surveys`](../src/platform/surveys/README.md)), and
`case` (the human-worked ticket; `CASE_STATUS_TRANSITIONS` support workflow, a `data` json field for the
collected bag — the [`worklist`](../src/platform/worklist/README.md) engine + `casework` preset).
A pack may **extend** one:

```yaml
entities:
  order:
    extends: system                 # merge over the platform template
    fields:
      gift_note: textarea           # pack-added — origin: pack, survives template evolution
    # + field_groups / list / label / relations. NOT pipeline / dedupe_by /
    #   title_field / system fields — the load-bearing shape is platform-owned
    #   (boot error). Need a different shape? Declare your own entity.
segments:
  unpaid_orders:                    # segments may target the (extended) system entity
    entity: order
    where: { field: payment_status, op: eq, value: pending }
```

Provenance rides the resolution (`ResolvedEntity.origin`/`extended`, per-field
`ResolvedField.origin`); redeclaring a system key *without* `extends: system`
is a boot error; `record` fields and relations may target system entities.
Orders are the shipped consumer — see
[`orders/README.md`](../src/platform/orders/README.md) and
[`docs/23-commerce.md`](23-commerce.md).

The shipped reference is the kaiian captain-onboarding pipeline —
[`src/packs/kaiian/xrm.yaml`](../src/packs/kaiian/xrm.yaml):

```yaml
entities:
  captain_application:
    label: { en: 'Captain application', ar: 'طلب انضمام كابتن' }
    # OPTIONAL display sections for the console record view (ordered; localized
    # headers). A field opts in via `group:`; ungrouped fields render in a trailing
    # section. Omit entirely → one flat section (legacy). Display-only.
    field_groups:
      - { key: application, label: { en: 'Application',          ar: 'الطلب' } }
      - { key: license,     label: { en: 'Driving license',      ar: 'رخصة القيادة' } }
      - { key: vehicle,     label: { en: 'Vehicle registration', ar: 'استمارة المركبة' } }
    fields:
      name:         { type: text, required: true, group: application }   # title_field defaults to `name`
      mobile:       { type: text, required: true, group: application }
      region:       { type: select, required: true, group: application, options: [central, western, eastern, northern, southern] }
      vehicle_type: { type: select, required: true, group: application, options: [sedan, suv, motorbike] }
      # OCR'd from the license photo (see become-captain `on_collect`); text so a
      # bad read can never fail the strict record_save. label shown in the console.
      id_number:    { group: license, label: { en: 'ID number', ar: 'رقم الهوية' } }   # bare-ish shorthand ≡ { type: text, … }
      nationality:  { group: license, label: { en: 'Nationality', ar: 'الجنسية' } }
      license_photo: { type: image, required: true, group: license, label: { en: 'License photo', ar: 'صورة الرخصة' } }
      # OCR'd from the vehicle registration photo.
      plate_no:     { group: vehicle, label: { en: 'Plate number', ar: 'رقم اللوحة' } }
      vehicle_make: { group: vehicle, label: { en: 'Make', ar: 'الصنع' } }
      notes:        textarea
    pipeline:
      stages:   [applied, docs_check, vehicle_check, training, active, rejected]
      terminal: [active, rejected]
      transitions:            # declared → strict funnel; OMIT for free movement
        applied:       [docs_check, rejected]
        docs_check:    [vehicle_check, rejected]
        vehicle_check: [training, rejected]
        training:      [active, rejected]
    list: [region, vehicle_type]   # composed into record_list subtitles + console columns
    dedupe_by: mobile              # record_save `match:` upserts on this field

segments:
  stalled_applications:
    entity: captain_application
    label: { en: 'Stalled 7d+' }
    where:
      all:
        - { stage_in: [applied, docs_check, vehicle_check, training] }
        - { not: { updated_within_days: 7 } }
```

Contract details (single source: `xrmFileSchema` +
[`contracts/spec.ts`](../src/platform/xrm/contracts/spec.ts)):

- **Field types**: `text · textarea · number · money · boolean · date ·
  select · multi_select · contact · record · image · image_list · document ·
  document_list · geo · json · line_items`. `document_list` is the multi-document
  sibling of `image_list` (an ordered array of document `MEDIA-` refs, cap 24 —
  uploaded paperwork/attachment sets; the console renders a download-link list). `money` stores the commerce `{ value, offset }`
  shape (`record_save` also accepts a plain decimal and normalises); `record`
  fields require `entity:` (a to-ONE ref, inline); to-MANY links are declared
  `relations:` (targets a declared entity or the literal `contact`). `image` /
  `document` store a `MEDIA-XXXXXXXX` ref (existence + kind checked against
  `media_assets` at save; the console record page shows a thumbnail / download
  link — see `docs/19-platform-media.md` §8b); **`image_list`** stores an
  ORDERED array of image refs (cap 24, each verified — the gallery/carousel
  source; the console renders a thumbnail grid). **`geo`** stores a
  range-validated `{ lat, lng }` point. Filters: presence (`is_set`/`not_set`)
  and **radius** — `{ field, op: within_km, value: { lat, lng, km } }` matches
  records within `km` of a point via an extension-free haversine (no PostGIS/
  earthdistance dependency). The console shows a map link. (Distance SORT — a
  "nearest-first" order — is still backlogged; the filter is the shipped half.)
- **Localized text** (`localized: true` on text/textarea): the value is ONE
  per-locale bag `{ en?, ar? }` (keys = `XRM_LOCALES`; strict, ≥1 locale, no
  bare-string coercion) — retiring the `name_en`/`name_ar` twin-field pattern.
  Search folds ALL locale values (either script fuzzy-matches);
  `eq`/`contains`/`in` filters match ANY locale; sort/group/title use the
  canonical `en ?? ar` value; flows render via the **`$loc(value)`** builtin
  (viewer-locale pick); the console shows every locale and edits with one
  input per locale. Excluded from `dedupe_by`/`match`.
- **Uniqueness** (`unique: true` on a scalar field, or the entity-level
  composite/scoped form `constraints: { unique: [{ fields: [a, b], where: … }] }`):
  duplicates among non-archived records of the entity are rejected with
  per-field reasons on the `failed` port. The optional `where:` narrows
  enforcement to rows matching a small, index-renderable predicate subset
  (`eq`/`in` on select/boolean/text fields + `stage`/`stage_in` + `all` — it
  becomes a partial-index predicate, so `now()`-relative nodes are out). ONE
  declaration enforces at TWO tiers: the app-side pre-check (friendly error)
  and a boot-provisioned partial UNIQUE index (race-safe — a concurrent
  duplicate maps back to the same error). See `uniqueConstraintSchema`
  ([`contracts/spec.ts`](../src/platform/xrm/contracts/spec.ts)) +
  [`xrm/index-ddl.ts`](../src/platform/xrm/index-ddl.ts); ops surface:
  `npm run db:xrm-indexes` (report / `--ensure` / `--prune`).
- **Public record pages** (`public:` on an entity): the platform serves an
  anonymous HTML page + JSON per record at `GET /r/:recordId`
  ([`routes/public-records.ts`](../src/routes/public-records.ts)) — the
  share-link successor to bespoke pack portals. Declare a `where:` visibility
  gate (outside it → 404, e.g. `{ stage: available }`), a strict `fields:`
  allowlist (contact-typed fields are a boot error — identity never
  serializes), and `expand:` record-ref paths (targets project as title +
  `list:` fields only). Flows mint absolute share links via the
  **`$record_url(record_id)`** builtin (base = `MEDIA_BASE_URL ?? PORTAL_URL`).
- **Pipeline defaults**: `initial` = first stage; omitted `transitions:` =
  **free movement** (declaring it opts into a gated funnel — a blocked move
  surfaces on `record_stage`'s `not_allowed` port with the legal next stages);
  `terminal:` defines "active" for `$xrm`, `record_list`'s default scope, and
  segments' `{ active: true }`.
- **Pipeline progression** (`progression: free | monotonic`, default `free`):
  `monotonic` makes the pipeline **forward-only by declared `stages` order** —
  any forward move (including skips) is allowed, a backward move returns the
  `not_allowed` outcome. It is the natural shape for application / onboarding
  funnels and journey-style progressions, and is **mutually exclusive** with a
  `transitions:` map (monotonic IS a transitions policy — declaring both is a
  boot error). See `progression` on `pipelineSchema` +
  `canTransitionStage`/`allowedNextStages` ([`pipeline.ts`](../src/platform/xrm/pipeline.ts)).
- **Milestones** (optional `milestones:` map): named, multi-completable
  checkpoints a record REACHES independently of its stage (a deal can be
  `quote_sent` and `site_visited` both at `negotiation`). Each declares an
  optional `label`, default `value` (added to the record's `milestone_value_sum`
  once, on first reach — the same rollup seam as line-item `rolls_up_to`),
  associated `stage` (analytics grouping), and `terminal: true` (reaching it
  completes the record — moves it into a terminal stage). `recordMilestone`
  appends a `milestone_reached` timeline event on EVERY reach (completion volume)
  and maintains the deduped `milestones` set + `milestone_value_sum` once per key.
  The generic engine behind journey goals, payments-received, docs-verified, KYC
  steps. See `milestoneSchema` + `resolveMilestones`
  ([`resolver.ts`](../src/platform/xrm/resolver.ts)) and `recordMilestone`
  ([`record-service.ts`](../src/platform/xrm/services/record-service.ts)).
- **Enrollment** (optional `enrollment:` block): the contact→record find-or-create
  + re-entry policy `resolveEnrollment` applies at write time (no background sweep).
  `key: contact` (v1); `reentry: none` (one run per contact for life — the default),
  `after_completion` (a fresh run once the prior completed + an entry signal +
  `cooldown_days`), or `anytime` (a fresh run on any entry signal past `cooldown_days`);
  `idle_reset_days` mints a fresh run when the latest was inactive longer than the
  window. `after_completion` requires a pipeline with a terminal stage (boot-checked).
  The marketing-automation re-entry standard, generic to any contact-keyed entity —
  repeat purchases, program re-enrollment, journey runs. Pairs with the new
  `xrm_records.completed_at` (set on entering a terminal stage). See `enrollmentSchema`
  + `resolveEnrollment` ([`record-service.ts`](../src/platform/xrm/services/record-service.ts)).
- **Funnel / conversion analytics** (`funnel-service.ts`, any pipelined entity):
  a stage-by-stage funnel (cumulative reached-≥-stage + drop-off + conversion %),
  milestone completion (volume vs unique reach + `total_value` + time-to-convert
  p50/p90), trends + first-touch cohorts, cost attribution (a `token_usage` join
  via the payload `conversation_id` — a conversation-level upper bound), and a
  currently-at drill-down (`listStageRecords`). All query-time over `xrm_events`
  (funnel order = the pipeline's `stages`); an optional `conversion_window_days`
  bounds how long after a run's first event a stage/milestone still counts.
  Generic — a kaiian `captain_application` acquisition funnel or an ecommerce
  cart→order funnel comes for free. See `getEntityFunnel` / `getMilestoneCompletion`
  / `getEntityOverview` ([`funnel-service.ts`](../src/platform/xrm/services/funnel-service.ts)).
- **Segments** are declared saved filters over one entity, compiled to
  parameterised SQL at boot (cap: 20/pack) and usable from flows
  (`record_list segment:`), the console segment browser (with live counts),
  and analytics.
- **Field grouping** (optional, display-only): declare an ordered `field_groups:`
  (each `{ key, label: {en,ar} }`) and tag fields with `group: <key>`. The console
  record view (`XrmRecordDetailPage`) renders one labeled section per group in that
  order, then a trailing untitled section for ungrouped fields; empty sections are
  omitted. Use it to organise a record by where its data came from (e.g. asked in
  chat vs OCR'd off each uploaded document — the kaiian captain application groups
  `application` / `license` / `vehicle`). Storage/validation are unaffected.
- Cross-field integrity is validated at boot (`validateXrmFile`) — unknown
  `title_field`, illegal op-for-type in a segment, an undeclared stage, a bad
  `default:`, a field `group:` with no matching `field_groups` key (or a duplicate
  key), etc. all fail pack load with a precise message.

## 3. Flow authoring recipes (the kaiian demos)

**Create a record conversationally** —
[`become-captain.flow.yaml`](../src/packs/kaiian/flows/tools/become-captain.flow.yaml):
`collect:` gathers the fields (pickers for `select` fields, validated free
text, an optional `required: false` enrichment field) → `use: approve_apply`
previews → `do: record_save` with `match: { value: $state.mobile }` (the
dedupe upsert — a returning applicant UPDATES, never duplicates) → on `ok`,
`$saved.created` gates a `task_open` (the 48h ops review task) and picks the
"received" vs "updated" copy. The primitive auto-links the conversation's
contact.

**Read status interactively** —
[`application-status.flow.yaml`](../src/packs/kaiian/flows/tools/application-status.flow.yaml):
`record_list` (contact-scoped by default; `active_only: false` so terminal
outcomes stay visible) → `empty` port → `answer:` (agent offers the intake
tool) · one/tapped → a detail card whose stage label + per-stage explainer
resolve via `$enum_label(stage, "stage" / "stage_hint")` from the pack locale ·
≥2 → `list_picker` whose rows re-invoke `$self` with `record_id`.

**Stage advancement** is operator-side in v1 (the console pipeline control
offers only legal moves). A flow may also call `record_stage` — branch on
`not_allowed` conversationally, and use `ok.terminal` to compose a journey
`track:` beside it (the flow-level bridge).

**Conventions that keep the agent grounded:**

- **memory_note shape** after a record write:
  `"<Entity> #<n> <verb> [→ <stage>] (record_id <uuid>)"` — the next-turn agent
  re-invokes tools with the concrete id, no list round-trip. Reads note what
  was shown.
- **`$xrm` binding**: when the pack declares xrm, every tool-flow turn gets
  `$xrm.<entity>.{count, latest.{record_id, record_number, title, stage,
  updated_at}}` over the contact's ACTIVE records (one bounded query) — use it
  for routing ("already has an open application → go straight to status")
  without a `record_list` call.
- **Agent prompt bullets** (see kaiian's
  [`identity.md`](../src/packs/kaiian/prompts/identity.md)): describe WHEN to
  call each tool + the anti-hallucination rule ("never state the stage from
  memory — call the tool").

## 4. Segment filter language (safety contract)

The `where:` AST is structured data, never an expression string. `compileWhere`
([`filters.ts`](../src/platform/xrm/filters.ts)) turns it into SQL under three
invariants: **values always bind as parameters**; the only spliced identifiers
are spec-validated, `^[a-z][a-z0-9_]*$`-constrained field/stage keys; ops are a
closed per-type table (illegal op → compile error at boot). `eq` compiles to
jsonb containment so the GIN `jsonb_path_ops` index on `fields` serves it;
money compares via the decimal (`value / offset`) expression; localized fields
admit `eq / in / contains / is_set / not_set`, compiled as any-locale ORs over
the closed `XRM_LOCALES` set; `geo` fields admit `is_set / not_set / within_km`
(the last matching `{ lat, lng, km }` via an extension-free haversine). Nodes:
field conds (`{ field, op, value }` — `eq ne gt gte lt lte in not_in contains
has is_set not_set within_km`), `{ stage }` / `{ stage_in }`, `{ active: true }`,
`{ created_within_days }` / `{ updated_within_days }`, and `all / any / not`
combinators. `record_list` accepts the same AST inline (`where:`), compiled per
call with the same guarantees — as do the console facet selections
(`filter=` on the records route) and the `public:` visibility gate.

## 5. Primitives · console · analytics

- **Primitives** (platform builtins, auto-discovered under
  `src/platform/primitives/xrm/`): `record_save · record_get · record_list ·
  record_search · record_related · record_aggregate · record_group ·
  record_stage · record_link · record_note · record_notify · task_open ·
  task_complete · task_list`. Full port contracts live in the generated
  [`16b-platform-stdlib-primitives.md`](16b-platform-stdlib-primitives.md) —
  never hand-listed here.
- **Relation expansion** (`expand:` on `record_get` / `record_list` /
  `record_search` / `record_related`): dot-paths over inline `record`/`contact`
  ref fields (depth ≤2, ≤8 paths — e.g. `[compound, compound.developer]`)
  resolve into a flat deduped **`refs` map** on the payload — one batched query
  per depth level, never per row ([`expand-service.ts`](../src/platform/xrm/services/expand-service.ts)).
  Rows keep raw ids; read the target via `refs[<id>]` (records carry their full
  `fields`; contacts a `{ display_name, channel_contact_handle }` summary).
  This retires the denormalised "snapshot label" workaround (clinic's old
  `branch_id`/`branch_label` pair is now a true `branch` record ref — retired).
- **Page envelopes.** `record_list`'s `ok`/`empty` payload is a page:
  `{ rows, total, offset, limit, has_more, next_offset, refs? }` — with
  `sort_by: { field, dir }` ordering by a declared field's VALUE
  (number/money/date/text/select, NULLS LAST) and `offset` paging.
  `record_search`/`record_related` return `{ rows, refs? }`.
- **Console (Customer OS)**: the "Customers" nav group — **Overview** (records
  by entity/stage + cases by queue + overdue tasks + segment sizes, each tile
  deep-linking), **Records** (entity/segment/stage-filtered inbox with a ranked
  **search box** for `search:`-declaring entities and **facet dropdowns with
  live counts** over every select/boolean field — selections compose a
  where-AST `filter=` param → record detail with a generically-rendered fields
  card incl. `image_list` galleries, localized values, geo map links, and
  ref fields as LINKS via the expand refs map, plus a **Related records** card
  (registry-derived reverse refs: every entity+field whose record ref points at
  the viewed record), pipeline control, timeline, links, tasks, note composer),
  **Cases** (the existing casework inbox, relocated into the group), **Tasks**
  (soonest-due inbox with inline complete). The contact detail page gains a
  tenant-wide **Records** card beside its Cases card. Backed by
  [`src/routes/xrm.ts`](../src/routes/xrm.ts); every GET degrades to
  `200 { has_xrm: false }` when the pack declares no xrm.
- **Permissions (RBAC).** The console surface is permission-scoped: a tenant
  `member` sees only the records their assigned pack roles grant (`view` on
  `record.<entity>`, scoped `own`/`team`/`queue`/`all`), an entity they hold no
  grant on doesn't appear in the catalog at all, and every write re-checks its verb
  (`update` / `transition` / `note` / `archive`) on that specific record. Lists,
  counts, search, and aggregates are narrowed by the same shared predicate
  (`ListRecordsFilter.workScope`). Operators + `owner`/`admin` bypass; agent/system
  callers (the `record_*` primitives, automation sweeps) pass no `access` and are
  unrestricted. See [`rbac/README.md`](../src/platform/rbac/README.md).
- **Analytics**: the XRM analytics service
  ([`analytics-service.ts`](../src/platform/xrm/services/analytics-service.ts))
  returns per-entity totals / stage distribution / observed stage flow /
  median stage dwell (computed from `xrm_events`), task throughput + overdue
  backlog, and live segment sizes — surfaced in the console Customer OS overview.

## 5b. Search & aggregation (`record_search` / `record_aggregate`)

XRM is a **searchable, aggregatable** object store — an entity opts in per-entity;
both capabilities are generic and domain-blind (they generalize what clinic
`doctor_search` and commerce `search_products`/`recommend_related` built per-domain —
the commerce `product` catalog now rides this generic surface). Schema:
migration [`059_xrm_search.sql`](../src/platform/db/migrations/059_xrm_search.sql)
(`pg_trgm` + `vector` on the platform DB, a maintained `search_text` + `embedding`
column on `xrm_records`, the `xrm_normalize` locale-folding helper).

- **`search:`** declaration on an entity (`contracts/spec.ts`):
  ```yaml
  entities:
    doctor:
      fields: { name: text, specialty: select, bio: textarea, rating: number }
      search: { text: [name, specialty], semantic: [name, bio], rank_by: rating }
  ```
  `text:` fields are folded (through `xrm_normalize` — Arabic letter/diacritic
  folding, so "احمد" matches "أحمد") into `search_text` for **fuzzy (pg_trgm)**
  matching. `semantic:` (opt-in, higher cost) embeds those fields on write
  (`saveRecord` calls the inference adapter; degrades to fuzzy-only if embedding is
  unavailable — never fails the write) for **pgvector** meaning-based ranking.
  `rank_by` is a numeric field used as the secondary sort. A caller may pass a
  **precomputed `embedding`** to `saveRecord` (baked author-time by `seed:gen`) to
  store it verbatim and skip the model call — catalog seeds apply with no inference.
- **`record_search { entity, query?, where?, contact_id?, limit?, expand? }`** —
  ranked records (fuzzy, blended with semantic cosine when the entity opts in),
  scored 0..1, then `rank_by`, then recency. Project-wide by default
  (reference/catalog entities have no owning contact); `where` is a facet filter
  (the segment AST). Payload `{ rows (w/ score), refs? }`.
  Ports: `ok` · `empty` (`rows: []`) · `failed`.
- **`record_related { entity, record_id, where?, contact_id?, limit?, expand? }`** —
  nearest-neighbour by an existing record's embedding ("customers also viewed"; the
  entity must declare `search.semantic`). A seed with no embedding → `empty`, never
  an error. Generalizes commerce `recommend_related`. Same payload shape as search.
- **`record_aggregate { entity, op, field?, where?, contact_id? }`** — a generic
  SCALAR rollup: `count` (no field) or `avg`/`sum`/`min`/`max`/`median` over a
  number/money field, `where`-filtered. Ports: `ok` (`{ value }`) · `failed`. The
  reusable rollup the surveys module composes (avg rating → resource `rank`).
- **`record_group { entity, by, metrics?, where?, contact_id?, sort?, limit?,
  expand_refs? }`** — GROUPED aggregation: `by` one dimension (a scalar field or
  `'stage'`), `count` always + named metrics (`sum/avg/min/max` on number/money,
  `distinct_count`, and **`only`** — the single distinct value when exactly one
  exists, else null: the "single compound in this city" pattern). `expand_refs`
  hydrates record/contact group keys + `only` values into a `refs` map. Payload
  `{ groups: [{ key, count, metrics }], refs? }` — the breakdown/facet-count
  engine (city→listing counts, category menus with entry prices, per-region
  dashboards). Ports: `ok` · `empty` · `failed`.

Full port contracts: [`16b-platform-stdlib-primitives.md`](16b-platform-stdlib-primitives.md).

## 5c. Customer-visible timeline + contact relay

The timeline is operator-facing by default, but any contact-linked record can also
carry a **customer** feed. `xrm_events` has a `visibility` (`operator` default /
`customer` / `internal`) + the `message_in` / `message_out` / `action` kinds.

- **`notifyRecordContact(recordId, { body, action?, memory_note? })`** — relays a
  message to the record's linked contact via `#platform/notify` and logs a
  `message_out` (`customer`) event with delivery status `{ notified, failure_reason? }`
  **written even when delivery fails** (so a failed relay shows a badge, never a
  silent gap). Message text is composed caller-side (`$t`). Flow primitive:
  **`record_notify`**. `appendRecordEvent` / `appendRecordMessage` write custom /
  message events directly. This is casework's mediated relay, generalized to every
  entity; the [`worklist`](../src/platform/worklist/README.md) `applyRecordAction`
  composes it for operator decisions.

## 6. Deferred (see `docs/BACKLOG.md`)

External-CRM sync adapter (seam: consume `xrm_events`) · cases-as-XRM-entity
evaluation (narrowed to dispositions/grammar — queues/SLA/assignment went
generic with the worklist absorption) · record dedupe/merge · scoring ·
CSV import/export · per-tenant expression indexes as a segment-scale escape
hatch (distinct from the SHIPPED catalog-driven per-entity indexes — §2
"Uniqueness" / `ensureXrmIndexes`). Shipped since first raised: campaigns
over segments (`#platform/automation`), per-operator ownership
(`assignee_principal`), notify-on-overdue-task.
