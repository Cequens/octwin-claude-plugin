# Platform Stdlib Primitives

> ⚙️ **Auto-generated** from `definePrimitive` metadata in `src/platform/primitives/*/primitive.ts`. Do NOT hand-edit — rerun `npm run platform-kb:platform-primitives` to regenerate. Source of truth is the TypeScript metadata; this page mirrors it for human consumption.

63 platform builtin primitives ship with the platform and are auto-discovered at boot. Every flow gets them for free — no per-pack wiring.

| Primitive | Purpose |
|---|---|
| [`analyze_image`](#analyze_image) | Analyze image(s) with a vision model via the platform Inference adapter. `image` is a URL or data URL; pass `images: [{image, mime_type}]` to analyze a batch in ONE call. `json: true` parses the output. Returns `{ text, json? }` on `ok`. |
| [`booking_cancel`](#booking_cancel) | Cancel a booking (XRM record): stage → cancelled + release the slot seat. Ports: ok · not_cancellable · failed. |
| [`booking_reschedule`](#booking_reschedule) | Move a booking (XRM record) to a new slot: claim the new seat, patch the record, release the old. Returns fresh reminders[].send_at. Ports: ok · full · invalid_slot · failed. |
| [`booking_reserve`](#booking_reserve) | Reserve a slot on a resource (an XRM record) — claims capacity race-free + writes the `booking` record. `fields` carries the pack's extended booking fields; `contact_id` defaults to the conversation contact. `ok` = { record_id, record_number, slot_start, reminders: [{ key, send_at }] } — pass reminders[].send_at to schedule_notify; `full` / `invalid_slot` carry an empty {}. Ports: ok · full · invalid_slot · failed. |
| [`campaign_create`](#campaign_create) | Create an outreach campaign (a `campaign` record at draft) over an entity segment / where-filter. Returns { campaign_id }; send with campaign_send. Ports: ok · failed. |
| [`campaign_send`](#campaign_send) | Send a campaign by id: resolve its audience + fan out one delivery per matched contact (via reminders), advancing it to `sent`. Returns { matched, enqueued, truncated, stage }. Ports: ok · failed. |
| [`cancel_scheduled`](#cancel_scheduled) | Cancel all pending scheduled messages for a `dedupe_key` (the handle passed to a prior schedule_notify). Returns how many were cancelled. |
| [`cart_add`](#cart_add) | Add a product (by retailer_id / SKU) to the contact's open cart, snapshotting its name/price/image from the catalog. Returns the updated cart view with precomputed totals. Ports: ok (CartView) · failed. |
| [`cart_clear`](#cart_clear) | Empty the contact's open cart. Ports: ok ({}) · failed. |
| [`cart_remove`](#cart_remove) | Remove a line from the contact's open cart. Returns the updated cart view. Ports: ok (CartView) · invalid (item not in cart) · failed. |
| [`cart_submit`](#cart_submit) | Submit the contact's open cart → create an order (reference_id) + mark the cart submitted. Returns the OrderView. Ports: ok (OrderView) · failed. |
| [`cart_update_qty`](#cart_update_qty) | Set a cart line's quantity (qty 0 removes it). Returns the updated cart view. Ports: ok (CartView) · invalid (item not in cart) · failed. |
| [`cart_view`](#cart_view) | Return the contact's open cart: line items + precomputed subtotal/totals. The CartView carries an `empty` flag. Ports: ok (CartView) · failed. |
| [`case_attach`](#case_attach) | Attach an uploaded media asset (a MEDIA- ref) to a case: link it (surfaces in the case's Attachments) and record it on the timeline. Use after case_open/case_reply when the customer sent evidence. Ports: ok ({ case_id, media_ref }) · failed. |
| [`case_get`](#case_get) | Fetch one case + its timeline by id. Ports: ok ({ case, events }) · empty (not found) · failed. |
| [`case_list`](#case_list) | List the current contact's support cases (optionally filtered by status). `ok` = an ARRAY of rows { id, case_number, type, status, priority, sla_due_at, created_at, updated_at, decision, decision_action, decision_params } (timestamps ISO strings; the decision* fields are null until a disposition is recorded); `empty` = []. Ports: ok · empty · failed. |
| [`case_open`](#case_open) | Open a support case from this conversation. `type` must be a declared case type; `fields` is the collected data bag. Queue + priority + SLA come from the pack's case-type declaration. Ports: ok ({ case_id, case_number, sla_hours, sla_due_at }) · failed. Use `case_number` (a per-project ticket number) + `sla_hours` to confirm the ticket to the customer. |
| [`case_reply`](#case_reply) | Record the customer's reply into their case: append it to the timeline (the 2nd-line team sees it) and, if the case was awaiting the customer, move it back to in_progress. Use this whenever the customer supplies information a case asked for. Ports: ok ({ case_id, status }) · failed. |
| [`case_transition`](#case_transition) | Move a case to a new status (workflow-validated). Ports: ok ({ status }) · failed (illegal transition / not found). |
| [`catalog_get`](#catalog_get) | Fetch one product from the platform catalog by retailer_id. Ports: ok (render-ready product row) · empty (null, not found) · failed. |
| [`catalog_list`](#catalog_list) | List products from the platform catalog with optional structured filters (category, brand, min_price/max_price in major units, availability), newest first. Returns render-ready rows. Ports: ok (rows) · empty ([]) · failed. |
| [`db_delete`](#db_delete) | Delete rows matching a structured `where` (REQUIRED, non-empty); returns the deleted count. |
| [`db_insert`](#db_insert) | Insert one row (object) or many (array) into a pack install table; the `ok` port returns the inserted rows array directly. |
| [`db_rpc`](#db_rpc) | Call a Postgres function (RPC) in the pack install schema. The function MUST be defined in a pack migration. Use for joins/aggregations/fuzzy matching that exceed a single-table db_select. Ports: `ok` (the RPC value directly — array of rows, or scalar/object — when non-empty), `empty` (no result — null or []), `failed`. A bare `bind:` binds the payload regardless; use `outputs:` to route the empty case without a follow-up if-check. |
| [`db_select`](#db_select) | Read rows from the pack install schema with a structured query (table, columns, where, order, limit, offset, single). Single-table reads; use db_rpc for joins/aggregations. Ports: `ok` (non-empty rows array, or the found row when `single`), `empty` (no result — [] rows / null single), `failed`. A bare `bind:` binds the payload regardless (`[]`/`null`); use `outputs:` to route the empty case without a follow-up if-check. |
| [`db_update`](#db_update) | Update rows matching a structured `where` (REQUIRED, non-empty) with `set`; returns the updated rows + count. |
| [`db_upsert`](#db_upsert) | Insert-or-update one/many rows (optionally on a unique constraint via `on_conflict`); the `ok` port returns the affected rows array directly. |
| [`embed_text`](#embed_text) | Embed a text string into a vector via the platform Inference adapter. Returns `{ embedding, dimensions }` on `ok`, where `embedding` is a pgvector text literal ready to bind to a `vector` column / RPC arg. |
| [`generate_text`](#generate_text) | One-shot LLM text generation via the platform Inference adapter. `model` defaults to GENERATE_MODEL. Returns `{ text, usage? }` on `ok`. |
| [`get_media`](#get_media) | Resolve a media asset by id or `MEDIA-XXXXXXXX` ref. Returns `{ url, mime_type, kind, caption, metadata }` on the `found` port — the URL is a fresh public/signed URL suitable for renderers or downstream fetch. |
| [`log_values`](#log_values) | Debug probe — emit a structured log line with the values the YAML wants to observe. Pure side-effect, no return data. Use to confirm an expression resolves correctly without adding a render or memory note. |
| [`notify`](#notify) | Channel-agnostic cross-user delivery. Resolves the recipient contact, selects the per-channel strategy, persists a `notifications` row, and returns one of: delivered / pending / failed. |
| [`order_get`](#order_get) | Fetch an order by reference_id. Ports: ok (OrderView) · empty (null) · failed. |
| [`order_list`](#order_list) | List orders (newest first), optionally scoped to a contact_id and filtered by status / payment_status. Ports: ok ({ orders, total }) · empty (no matches) · failed. |
| [`order_update_status`](#order_update_status) | Update an order's fulfillment + payment status by reference_id. Ports: ok (OrderView) · empty (null) · failed. |
| [`payment_request`](#payment_request) | Build the payment rail for an order checkout (v1: payment_link). Returns { payment_url, rail, reference_id } for embedding in an order_details render. Ports: ok · failed. |
| [`product_search`](#product_search) | Semantic + filtered product search over the platform catalog. Embeds `query` and ranks by similarity, combined with optional structured filters (category, brand, min_price/max_price in major units, availability). Returns render-ready rows ({ retailer_id, title, price, price_minor, currency, image_url, description, similarity }). Ports: ok (rows) · empty ([]) · failed. |
| [`recommend_related`](#recommend_related) | Related/upsell products (nearest embedding neighbours to retailer_id). Returns render-ready rows. Ports: ok (rows) · empty ([]) · failed. |
| [`record_aggregate`](#record_aggregate) | Aggregate a where-filtered record set: `count` (no field) or a numeric op (avg/sum/min/max/median) over a number/money `field`. Project-wide by default; pass `contact_id` to scope. Ports: ok ({ value }) · failed. |
| [`record_get`](#record_get) | Fetch one XRM record by `record_id`, or by `entity` + `match: { field?, value }` (field defaults to the entity's `dedupe_by`). `expand: [ref_field, ref_field.ref_field]` resolves record/contact ref fields into a `refs` map. Ports: ok (record + refs?) · not_found · failed. |
| [`record_group`](#record_group) | Group an entity's records by one dimension (`by`: a scalar field or 'stage') with per-group metrics: count (always) + named `metrics` ({ op: sum|avg|min|max|distinct_count|only, field }). Optional `where:` AST, `contact_id`, `sort: { by: count|key|<metric>, dir }`, `limit`, `expand_refs` (resolve record/contact keys + only-values into `refs`). Payload: { groups: [{ key, count, metrics }], refs? }. Ports: ok · empty · failed. |
| [`record_link`](#record_link) | Add (or with `remove: true`, remove) a to-many link on a declared relation — exactly one of `to_record_id` / `to_contact_id`. Idempotent. Ports: ok ({ record_id, relation, changed }) · not_found · failed. |
| [`record_list`](#record_list) | List XRM records as a page envelope { rows, total, offset, limit, has_more, next_offset, refs? } — each row is { id, record_number, title, subtitle, stage, fields, updated_at } (read your declared field values off `row.fields.<name>`). Pass a declared `segment` (project-wide), or an `entity` (scoped to the current contact unless `all: true` / `contact_id`). Optional inline `where:` AST, `stage`, `sort_by: { field, dir }` (field-value ordering), `offset`, `limit`, `expand: [ref_field, …]`. Ports: ok · empty (rows: []) · failed. |
| [`record_note`](#record_note) | Append a note to an XRM record's timeline. Ports: ok ({ record_id }) · not_found · failed. |
| [`record_notify`](#record_notify) | Relay a message to a record's linked contact (via notify) and log a customer-visible message_out timeline event with delivery status — written even on failure. Ports: ok ({ notified, failure_reason? }) · failed. |
| [`record_related`](#record_related) | Records most similar to `record_id` by semantic embedding ("also viewed" / related items). The entity must declare `search.semantic` in xrm.yaml. Project-wide by default; pass `contact_id` to scope, `where:` to facet, `limit`, `expand: [ref_field, …]`. Payload: { rows (ranked, w/ score), refs? }. Ports: ok · empty (rows: []) · failed. |
| [`record_save`](#record_save) | Create or update an XRM record. `entity` must be a declared entity; `fields` validates against its spec. Pass `record_id` to update, or `match: { value }` for a dedupe upsert on the entity's `dedupe_by` field. Ports: ok ({ record_id, record_number, entity, title, stage, created }) · invalid ({ reason, fields? } — per-field validation, agent fixes + retries) · failed ({ reason } — system). |
| [`record_search`](#record_search) | Search an entity's records by relevance to `query` (fuzzy; semantic when the entity opts in). Project-wide by default; pass `contact_id` to scope. Optional `where:` facet filter, `active_only`, `limit`, `expand: [ref_field, …]` (resolves ref fields into `refs`). The entity must declare `search:` in xrm.yaml. Payload: { rows (ranked, w/ score), refs? }. Ports: ok · empty (rows: []) · failed. |
| [`record_stage`](#record_stage) | Move an XRM record to pipeline stage `to` (optionally with a timeline `note`). Ports: ok ({ record_id, from, to, terminal }) · not_allowed ({ from, to, allowed — offer these instead }) · not_found · failed. |
| [`resolve_contact_by_metadata`](#resolve_contact_by_metadata) | Look up a platform contact by a namespaced (namespace, field, value) triple within the current tenant. Fallback path for packs that do not store the contact id on their own user row. Returns ok / not_found / failed. |
| [`schedule_notify`](#schedule_notify) | Schedule a notify for later delivery (deliver `intent` to a contact at `send_at`). Use `dedupe_key` for idempotency + as a handle to later `cancel_scheduled`. The intent (incl. any button taps) is delivered verbatim. |
| [`send_message`](#send_message) | Send a text or rendered intent to the active channel mid-flow. Distinct from `render:` (which emits ONE response per step) — use `send_message` when a step needs to emit multiple messages, e.g. progress updates. |
| [`set_media_metadata`](#set_media_metadata) | Merge a JSONB patch into a media asset's `metadata` column. Used by pack media-handler flows to attach analysis results (vision output, OCR text, …) to the asset record. Top-level keys replace wholesale (jsonb || semantics). |
| [`slot_list`](#slot_list) | Free future bookable slots for a resource (an XRM record) over a date window. `ok` is a BARE ARRAY of rows (no wrapper — bind it and iterate directly), each row { slot_start, slot_end, slot_minutes, remaining, capacity, dow, local_date, local_time, meta }; `empty` is []. `meta` is the rule's opaque bag (e.g. a per-day branch); compose the label pack-side. Ports: ok · empty · failed. |
| [`storage_remove`](#storage_remove) | Delete bucket/path from the pack install bucket. |
| [`survey_submit`](#survey_submit) | Record a survey response: validate `answers` against the declared `survey`, write a survey_response XRM record (upsert on `subject_ref`), compute the score. `contact_id` defaults to the conversation contact. Ports: ok ({ record_id, score }) · failed. |
| [`task_complete`](#task_complete) | Close an open XRM task as done (default) or cancelled. Ports: ok ({ task_id, status }) · not_found (unknown / already closed) · failed. |
| [`task_list`](#task_list) | List XRM follow-up tasks for a record or contact (default: the conversation's contact), soonest-due first. `overdue_only: true` narrows to overdue open tasks. Ports: ok (rows) · empty · failed. |
| [`task_open`](#task_open) | Open an operator follow-up task (optionally on a record; contact defaults to the conversation's). Due time: `due_at` (ISO) or `due_in_hours` — not both. Ports: ok ({ task_id, due_at }) · failed. |
| [`track_event`](#track_event) | Emit a customer-journey signal (one of event / stage_enter / goal_reached). An `event` fans out into its declared stage/goal transitions. `journey:` is only needed when the id is ambiguous across the pack's journeys. |
| [`upsert_platform_contact`](#upsert_platform_contact) | Overlay pack-domain details onto the platform `contacts` row for (tenant, channel, channel_contact_handle). Used inside a pre-turn flow AFTER lookup_or_create_pack_user to layer pack name / language / namespaced pack_metadata onto the platform's baseline upsert. Idempotent (ON CONFLICT DO UPDATE); JSONB metadata deep-merges per namespace. |
| [`work_action`](#work_action) | Apply a declared operator action to a worked record: records the decision, optionally transitions the stage (action.to_stage), optionally relays `relay_body` to the record's contact (when the action declares relay). Ports: ok (ApplyActionResult) · failed. |
| [`work_open`](#work_open) | Open a worked record: save an XRM record of `entity` (fields incl. any sub-type discriminator) and enroll it into its routed worklist queue. `contact_id` defaults to the conversation contact. Returns { record_id, record_number, queue_key } (queue_key null = unrouted). Ports: ok · failed. |

Companion catalogs:
- Machine-readable JSON: [`dist/octwin-platform-kb/platform-primitives.json`](../dist/octwin-platform-kb/platform-primitives.json)

---

## `analyze_image`

Analyze image(s) with a vision model via the platform Inference adapter. `image` is a URL or data URL; pass `images: [{image, mime_type}]` to analyze a batch in ONE call. `json: true` parses the output. Returns `{ text, json? }` on `ok`.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | string |  |  |
| `mime_type` | string |  |  |
| `images` | array&lt;object&gt; |  |  |
| `prompt` | string | ✓ |  |
| `json` | boolean |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | ✓ |  |
| `json` | any |  |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `booking_cancel`

Cancel a booking (XRM record): stage → cancelled + release the slot seat. Ports: ok · not_cancellable · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `not_cancellable`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `booking_reschedule`

Move a booking (XRM record) to a new slot: claim the new seat, patch the record, release the old. Returns fresh reminders[].send_at. Ports: ok · full · invalid_slot · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `new_slot_start` | string | ✓ |  |
| `slot_minutes` | integer |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `full`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `invalid_slot`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `booking_reserve`

Reserve a slot on a resource (an XRM record) — claims capacity race-free + writes the `booking` record. `fields` carries the pack's extended booking fields; `contact_id` defaults to the conversation contact. `ok` = { record_id, record_number, slot_start, reminders: [{ key, send_at }] } — pass reminders[].send_at to schedule_notify; `full` / `invalid_slot` carry an empty {}. Ports: ok · full · invalid_slot · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `resource_record_id` | string | ✓ |  |
| `slot_start` | string | ✓ |  |
| `slot_minutes` | integer |  |  |
| `fields` | object |  |  |
| `contact_id` | string |  |  |
| `initial_stage` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `full`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `invalid_slot`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `campaign_create`

Create an outreach campaign (a `campaign` record at draft) over an entity segment / where-filter. Returns { campaign_id }; send with campaign_send. Ports: ok · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✓ |  |
| `entity` | string | ✓ |  |
| `message` | string | ✓ |  |
| `segment_key` | string |  |  |
| `where` | any |  |  |
| `channel` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `campaign_id` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `campaign_send`

Send a campaign by id: resolve its audience + fan out one delivery per matched contact (via reminders), advancing it to `sent`. Returns { matched, enqueued, truncated, stage }. Ports: ok · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `campaign_id` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `matched` | number | ✓ |  |
| `enqueued` | number | ✓ |  |
| `truncated` | boolean | ✓ |  |
| `stage` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cancel_scheduled`

Cancel all pending scheduled messages for a `dedupe_key` (the handle passed to a prior schedule_notify). Returns how many were cancelled.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `dedupe_key` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `cancelled` | number | ✓ |  |

---

## `cart_add`

Add a product (by retailer_id / SKU) to the contact's open cart, snapshotting its name/price/image from the catalog. Returns the updated cart view with precomputed totals. Ports: ok (CartView) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |
| `product_id` | string | ✓ |  |
| `qty` | number |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `invalid`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cart_clear`

Empty the contact's open cart. Ports: ok ({}) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cart_remove`

Remove a line from the contact's open cart. Returns the updated cart view. Ports: ok (CartView) · invalid (item not in cart) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |
| `retailer_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `invalid`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cart_submit`

Submit the contact's open cart → create an order (reference_id) + mark the cart submitted. Returns the OrderView. Ports: ok (OrderView) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |
| `note` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cart_update_qty`

Set a cart line's quantity (qty 0 removes it). Returns the updated cart view. Ports: ok (CartView) · invalid (item not in cart) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |
| `retailer_id` | string | ✓ |  |
| `qty` | number | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `invalid`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `cart_view`

Return the contact's open cart: line items + precomputed subtotal/totals. The CartView carries an `empty` flag. Ports: ok (CartView) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_attach`

Attach an uploaded media asset (a MEDIA- ref) to a case: link it (surfaces in the case's Attachments) and record it on the timeline. Use after case_open/case_reply when the customer sent evidence. Ports: ok ({ case_id, media_ref }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `media_id` | string | ✓ |  |
| `caption` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `media_ref` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_get`

Fetch one case + its timeline by id. Ports: ok ({ case, events }) · empty (not found) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_list`

List the current contact's support cases (optionally filtered by status). `ok` = an ARRAY of rows { id, case_number, type, status, priority, sla_due_at, created_at, updated_at, decision, decision_action, decision_params } (timestamps ISO strings; the decision* fields are null until a disposition is recorded); `empty` = []. Ports: ok · empty · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string |  |  |
| `status` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_open`

Open a support case from this conversation. `type` must be a declared case type; `fields` is the collected data bag. Queue + priority + SLA come from the pack's case-type declaration. Ports: ok ({ case_id, case_number, sla_hours, sla_due_at }) · failed. Use `case_number` (a per-project ticket number) + `sla_hours` to confirm the ticket to the customer.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | ✓ |  |
| `fields` | object |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `case_number` | number | ✓ |  |
| `sla_hours` | union | ✓ |  |
| `sla_due_at` | union | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_reply`

Record the customer's reply into their case: append it to the timeline (the 2nd-line team sees it) and, if the case was awaiting the customer, move it back to in_progress. Use this whenever the customer supplies information a case asked for. Ports: ok ({ case_id, status }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `message` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `status` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `case_transition`

Move a case to a new status (workflow-validated). Ports: ok ({ status }) · failed (illegal transition / not found).

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `case_id` | string | ✓ |  |
| `to_status` | string | ✓ |  |
| `note` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `catalog_get`

Fetch one product from the platform catalog by retailer_id. Ports: ok (render-ready product row) · empty (null, not found) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `retailer_id` | string | ✓ |  |
| `locale` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `catalog_list`

List products from the platform catalog with optional structured filters (category, brand, min_price/max_price in major units, availability), newest first. Returns render-ready rows. Ports: ok (rows) · empty ([]) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `category` | string |  |  |
| `brand` | string |  |  |
| `min_price` | number |  |  |
| `max_price` | number |  |  |
| `availability` | string |  |  |
| `limit` | number |  |  |
| `locale` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_delete`

Delete rows matching a structured `where` (REQUIRED, non-empty); returns the deleted count.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `table` | string | ✓ |  |
| `where` | any | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `count` | number | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_insert`

Insert one row (object) or many (array) into a pack install table; the `ok` port returns the inserted rows array directly.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `table` | string | ✓ |  |
| `values` | union | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_rpc`

Call a Postgres function (RPC) in the pack install schema. The function MUST be defined in a pack migration. Use for joins/aggregations/fuzzy matching that exceed a single-table db_select. Ports: `ok` (the RPC value directly — array of rows, or scalar/object — when non-empty), `empty` (no result — null or []), `failed`. A bare `bind:` binds the payload regardless; use `outputs:` to route the empty case without a follow-up if-check.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `fn` | string | ✓ |  |
| `params` | object |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_select`

Read rows from the pack install schema with a structured query (table, columns, where, order, limit, offset, single). Single-table reads; use db_rpc for joins/aggregations. Ports: `ok` (non-empty rows array, or the found row when `single`), `empty` (no result — [] rows / null single), `failed`. A bare `bind:` binds the payload regardless (`[]`/`null`); use `outputs:` to route the empty case without a follow-up if-check.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `table` | string | ✓ |  |
| `columns` | string |  |  |
| `where` | any |  |  |
| `order` | any |  |  |
| `limit` | integer |  |  |
| `offset` | integer |  |  |
| `single` | boolean |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_update`

Update rows matching a structured `where` (REQUIRED, non-empty) with `set`; returns the updated rows + count.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `table` | string | ✓ |  |
| `where` | any | ✓ |  |
| `set` | object | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `rows` | array&lt;any&gt; | ✓ |  |
| `count` | number | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `db_upsert`

Insert-or-update one/many rows (optionally on a unique constraint via `on_conflict`); the `ok` port returns the affected rows array directly.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `table` | string | ✓ |  |
| `values` | union | ✓ |  |
| `on_conflict` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `embed_text`

Embed a text string into a vector via the platform Inference adapter. Returns `{ embedding, dimensions }` on `ok`, where `embedding` is a pgvector text literal ready to bind to a `vector` column / RPC arg.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `embedding` | string | ✓ |  |
| `dimensions` | number | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `generate_text`

One-shot LLM text generation via the platform Inference adapter. `model` defaults to GENERATE_MODEL. Returns `{ text, usage? }` on `ok`.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | ✓ |  |
| `model` | string |  |  |
| `system` | string |  |  |
| `temperature` | number |  |  |
| `max_tokens` | number |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | ✓ |  |
| `usage` | object |  |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `get_media`

Resolve a media asset by id or `MEDIA-XXXXXXXX` ref. Returns `{ url, mime_type, kind, caption, metadata }` on the `found` port — the URL is a fresh public/signed URL suitable for renderers or downstream fetch.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `media_id` | string | ✓ |  |

### Output ports

#### `found`

| Field | Type | Required | Description |
|---|---|---|---|
| `media_id` | string | ✓ |  |
| `kind` | enum: `image` \| `document` | ✓ |  |
| `mime_type` | string | ✓ |  |
| `caption` | union | ✓ |  |
| `url` | string | ✓ |  |
| `metadata` | object | ✓ |  |

#### `not_found`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `log_values`

Debug probe — emit a structured log line with the values the YAML wants to observe. Pure side-effect, no return data. Use to confirm an expression resolves correctly without adding a render or memory note.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `label` | string | ✓ |  |
| `values` | object | ✓ |  |
| `level` | enum: `info` \| `warn` \| `error` |  |  |

### Output ports

#### `ok`

_(empty object — primitive takes/returns `{}`)_

---

## `notify`

Channel-agnostic cross-user delivery. Resolves the recipient contact, selects the per-channel strategy, persists a `notifications` row, and returns one of: delivered / pending / failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `to` | union | ✓ |  |
| `intent` | any | ✓ |  |
| `memory_note` | string |  |  |
| `visibility` | enum: `permanent` \| `ephemeral` \| `diagnostic` |  |  |
| `echo_delivery` | boolean |  |  |

### Output ports

#### `delivered`

| Field | Type | Required | Description |
|---|---|---|---|
| `notification_id` | string | ✓ |  |
| `provider_message_id` | string |  |  |

#### `pending`

| Field | Type | Required | Description |
|---|---|---|---|
| `notification_id` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `notification_id` | string | ✓ |  |
| `reason` | string | ✓ |  |

---

## `order_get`

Fetch an order by reference_id. Ports: ok (OrderView) · empty (null) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `reference_id` | string | ✓ |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `order_list`

List orders (newest first), optionally scoped to a contact_id and filtered by status / payment_status. Ports: ok ({ orders, total }) · empty (no matches) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string |  |  |
| `status` | string |  |  |
| `payment_status` | string |  |  |
| `limit` | number |  |  |
| `offset` | number |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `order_update_status`

Update an order's fulfillment + payment status by reference_id. Ports: ok (OrderView) · empty (null) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `reference_id` | string | ✓ |  |
| `status` | string |  |  |
| `payment_status` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `payment_request`

Build the payment rail for an order checkout (v1: payment_link). Returns { payment_url, rail, reference_id } for embedding in an order_details render. Ports: ok · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `reference_id` | string | ✓ |  |
| `checkout_base_url` | string |  |  |
| `rail` | enum: `payment_link` \| `payment_gateway` \| `pix` \| `boleto` |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `product_search`

Semantic + filtered product search over the platform catalog. Embeds `query` and ranks by similarity, combined with optional structured filters (category, brand, min_price/max_price in major units, availability). Returns render-ready rows ({ retailer_id, title, price, price_minor, currency, image_url, description, similarity }). Ports: ok (rows) · empty ([]) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `query` | string |  |  |
| `category` | string |  |  |
| `brand` | string |  |  |
| `min_price` | number |  |  |
| `max_price` | number |  |  |
| `availability` | string |  |  |
| `limit` | number |  |  |
| `locale` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `recommend_related`

Related/upsell products (nearest embedding neighbours to retailer_id). Returns render-ready rows. Ports: ok (rows) · empty ([]) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `retailer_id` | string | ✓ |  |
| `limit` | number |  |  |
| `locale` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {}
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_aggregate`

Aggregate a where-filtered record set: `count` (no field) or a numeric op (avg/sum/min/max/median) over a number/money `field`. Project-wide by default; pass `contact_id` to scope. Ports: ok ({ value }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `op` | enum: `count` \| `avg` \| `sum` \| `min` \| `max` \| `median` | ✓ |  |
| `field` | string |  |  |
| `where` | any |  |  |
| `contact_id` | string |  |  |
| `active_only` | boolean |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | union | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_get`

Fetch one XRM record by `record_id`, or by `entity` + `match: { field?, value }` (field defaults to the entity's `dedupe_by`). `expand: [ref_field, ref_field.ref_field]` resolves record/contact ref fields into a `refs` map. Ports: ok (record + refs?) · not_found · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string |  |  |
| `entity` | string |  |  |
| `match` | object |  |  |
| `expand` | array&lt;string&gt; |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✓ |  |
| `entity` | string | ✓ |  |
| `record_number` | number | ✓ |  |
| `title` | union | ✓ |  |
| `stage` | union | ✓ |  |
| `contact_id` | union | ✓ |  |
| `fields` | object | ✓ |  |
| `created_at` | string | ✓ |  |
| `updated_at` | string | ✓ |  |
| `refs` | object |  |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_group`

Group an entity's records by one dimension (`by`: a scalar field or 'stage') with per-group metrics: count (always) + named `metrics` ({ op: sum|avg|min|max|distinct_count|only, field }). Optional `where:` AST, `contact_id`, `sort: { by: count|key|<metric>, dir }`, `limit`, `expand_refs` (resolve record/contact keys + only-values into `refs`). Payload: { groups: [{ key, count, metrics }], refs? }. Ports: ok · empty · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `by` | string | ✓ |  |
| `metrics` | object |  |  |
| `where` | any |  |  |
| `contact_id` | string |  |  |
| `active_only` | boolean |  |  |
| `sort` | object |  |  |
| `limit` | integer |  |  |
| `expand_refs` | boolean |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_link`

Add (or with `remove: true`, remove) a to-many link on a declared relation — exactly one of `to_record_id` / `to_contact_id`. Idempotent. Ports: ok ({ record_id, relation, changed }) · not_found · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `relation` | string | ✓ |  |
| `to_record_id` | string |  |  |
| `to_contact_id` | string |  |  |
| `remove` | boolean |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `relation` | string | ✓ |  |
| `changed` | boolean | ✓ |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_list`

List XRM records as a page envelope { rows, total, offset, limit, has_more, next_offset, refs? } — each row is { id, record_number, title, subtitle, stage, fields, updated_at } (read your declared field values off `row.fields.<name>`). Pass a declared `segment` (project-wide), or an `entity` (scoped to the current contact unless `all: true` / `contact_id`). Optional inline `where:` AST, `stage`, `sort_by: { field, dir }` (field-value ordering), `offset`, `limit`, `expand: [ref_field, …]`. Ports: ok · empty (rows: []) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string |  |  |
| `segment` | string |  |  |
| `where` | any |  |  |
| `contact_id` | string |  |  |
| `all` | boolean |  |  |
| `stage` | string |  |  |
| `active_only` | boolean |  |  |
| `limit` | integer |  |  |
| `offset` | integer |  |  |
| `sort` | enum: `updated_at` \| `created_at` \| `record_number` \| `stage` |  |  |
| `sort_by` | object |  |  |
| `expand` | array&lt;string&gt; |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_note`

Append a note to an XRM record's timeline. Ports: ok ({ record_id }) · not_found · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `body` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_notify`

Relay a message to a record's linked contact (via notify) and log a customer-visible message_out timeline event with delivery status — written even on failure. Ports: ok ({ notified, failure_reason? }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `body` | string | ✓ |  |
| `action` | string |  |  |
| `memory_note` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `notified` | boolean | ✓ |  |
| `failure_reason` | string |  |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_related`

Records most similar to `record_id` by semantic embedding ("also viewed" / related items). The entity must declare `search.semantic` in xrm.yaml. Project-wide by default; pass `contact_id` to scope, `where:` to facet, `limit`, `expand: [ref_field, …]`. Payload: { rows (ranked, w/ score), refs? }. Ports: ok · empty (rows: []) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `record_id` | string | ✓ |  |
| `where` | any |  |  |
| `contact_id` | string |  |  |
| `active_only` | boolean |  |  |
| `limit` | integer |  |  |
| `expand` | array&lt;string&gt; |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_save`

Create or update an XRM record. `entity` must be a declared entity; `fields` validates against its spec. Pass `record_id` to update, or `match: { value }` for a dedupe upsert on the entity's `dedupe_by` field. Ports: ok ({ record_id, record_number, entity, title, stage, created }) · invalid ({ reason, fields? } — per-field validation, agent fixes + retries) · failed ({ reason } — system).

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `fields` | object |  |  |
| `record_id` | string |  |  |
| `match` | object |  |  |
| `stage` | string |  |  |
| `contact_id` | string |  |  |
| `no_contact` | boolean |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `record_number` | number | ✓ |  |
| `entity` | string | ✓ |  |
| `title` | union | ✓ |  |
| `stage` | union | ✓ |  |
| `created` | boolean | ✓ |  |

#### `invalid`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |
| `fields` | array&lt;object&gt; |  |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_search`

Search an entity's records by relevance to `query` (fuzzy; semantic when the entity opts in). Project-wide by default; pass `contact_id` to scope. Optional `where:` facet filter, `active_only`, `limit`, `expand: [ref_field, …]` (resolves ref fields into `refs`). The entity must declare `search:` in xrm.yaml. Payload: { rows (ranked, w/ score), refs? }. Ports: ok · empty (rows: []) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `query` | string |  |  |
| `where` | any |  |  |
| `contact_id` | string |  |  |
| `active_only` | boolean |  |  |
| `limit` | integer |  |  |
| `expand` | array&lt;string&gt; |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `record_stage`

Move an XRM record to pipeline stage `to` (optionally with a timeline `note`). Ports: ok ({ record_id, from, to, terminal }) · not_allowed ({ from, to, allowed — offer these instead }) · not_found · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `to` | string | ✓ |  |
| `note` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `from` | string | ✓ |  |
| `to` | string | ✓ |  |
| `terminal` | boolean | ✓ |  |

#### `not_allowed`

| Field | Type | Required | Description |
|---|---|---|---|
| `from` | string | ✓ |  |
| `to` | string | ✓ |  |
| `allowed` | array&lt;string&gt; | ✓ |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `resolve_contact_by_metadata`

Look up a platform contact by a namespaced (namespace, field, value) triple within the current tenant. Fallback path for packs that do not store the contact id on their own user row. Returns ok / not_found / failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `namespace` | string | ✓ |  |
| `field` | string | ✓ |  |
| `value` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |
| `channel` | string | ✓ |  |
| `channel_contact_handle` | string | ✓ |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `schedule_notify`

Schedule a notify for later delivery (deliver `intent` to a contact at `send_at`). Use `dedupe_key` for idempotency + as a handle to later `cancel_scheduled`. The intent (incl. any button taps) is delivered verbatim.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `to` | object | ✓ |  |
| `send_at` | string | ✓ |  |
| `intent` | any | ✓ |  |
| `memory_note` | string |  |  |
| `dedupe_key` | string |  |  |

### Output ports

#### `scheduled`

| Field | Type | Required | Description |
|---|---|---|---|
| `scheduled_id` | string |  |  |
| `deduped` | boolean | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `send_message`

Send a text or rendered intent to the active channel mid-flow. Distinct from `render:` (which emits ONE response per step) — use `send_message` when a step needs to emit multiple messages, e.g. progress updates.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `to` | string | ✓ |  |
| `text` | string |  |  |
| `intent` | any |  |  |

### Output ports

#### `delivered`

| Field | Type | Required | Description |
|---|---|---|---|
| `provider_message_id` | string |  |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `set_media_metadata`

Merge a JSONB patch into a media asset's `metadata` column. Used by pack media-handler flows to attach analysis results (vision output, OCR text, …) to the asset record. Top-level keys replace wholesale (jsonb || semantics).

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `media_id` | string | ✓ |  |
| `patch` | object | ✓ |  |

### Output ports

#### `ok`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `slot_list`

Free future bookable slots for a resource (an XRM record) over a date window. `ok` is a BARE ARRAY of rows (no wrapper — bind it and iterate directly), each row { slot_start, slot_end, slot_minutes, remaining, capacity, dow, local_date, local_time, meta }; `empty` is []. `meta` is the rule's opaque bag (e.g. a per-day branch); compose the label pack-side. Ports: ok · empty · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `resource_record_id` | string | ✓ |  |
| `from` | string |  |  |
| `days` | integer |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `storage_remove`

Delete bucket/path from the pack install bucket.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `bucket` | string | ✓ |  |
| `path` | string | ✓ |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `ok` | boolean | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `survey_submit`

Record a survey response: validate `answers` against the declared `survey`, write a survey_response XRM record (upsert on `subject_ref`), compute the score. `contact_id` defaults to the conversation contact. Ports: ok ({ record_id, score }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `survey` | string | ✓ |  |
| `answers` | object |  |  |
| `subject_ref` | string |  |  |
| `contact_id` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `score` | union | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `task_complete`

Close an open XRM task as done (default) or cancelled. Ports: ok ({ task_id, status }) · not_found (unknown / already closed) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | ✓ |  |
| `outcome` | enum: `done` \| `cancelled` |  |  |
| `note` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | ✓ |  |
| `status` | string | ✓ |  |

#### `not_found`

_(empty object — primitive takes/returns `{}`)_

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `task_list`

List XRM follow-up tasks for a record or contact (default: the conversation's contact), soonest-due first. `overdue_only: true` narrows to overdue open tasks. Ports: ok (rows) · empty · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string |  |  |
| `contact_id` | string |  |  |
| `status` | enum: `open` \| `done` \| `cancelled` |  |  |
| `overdue_only` | boolean |  |  |
| `limit` | integer |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `empty`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `task_open`

Open an operator follow-up task (optionally on a record; contact defaults to the conversation's). Due time: `due_at` (ISO) or `due_in_hours` — not both. Ports: ok ({ task_id, due_at }) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | ✓ |  |
| `body` | string |  |  |
| `record_id` | string |  |  |
| `contact_id` | string |  |  |
| `due_at` | string |  |  |
| `due_in_hours` | number |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | ✓ |  |
| `due_at` | union | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `track_event`

Emit a customer-journey signal (one of event / stage_enter / goal_reached). An `event` fans out into its declared stage/goal transitions. `journey:` is only needed when the id is ambiguous across the pack's journeys.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `journey` | string |  |  |
| `event` | string |  |  |
| `stage_enter` | string |  |  |
| `goal_reached` | string |  |  |
| `value` | number |  |  |
| `metadata` | object |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `written` | number | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `upsert_platform_contact`

Overlay pack-domain details onto the platform `contacts` row for (tenant, channel, channel_contact_handle). Used inside a pre-turn flow AFTER lookup_or_create_pack_user to layer pack name / language / namespaced pack_metadata onto the platform's baseline upsert. Idempotent (ON CONFLICT DO UPDATE); JSONB metadata deep-merges per namespace.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `channel_contact_handle` | string | ✓ |  |
| `channel` | enum: `whatsapp` \| `slack` \| `web` \| `voice` \| `sms` |  |  |
| `display_name` | union |  |  |
| `language` | union |  |  |
| `pack_metadata` | object |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `contact_id` | string | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `work_action`

Apply a declared operator action to a worked record: records the decision, optionally transitions the stage (action.to_stage), optionally relays `relay_body` to the record's contact (when the action declares relay). Ports: ok (ApplyActionResult) · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `record_id` | string | ✓ |  |
| `action` | string | ✓ |  |
| `params` | object |  |  |
| `relay_body` | string |  |  |

### Output ports

#### `ok`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema"
}
```

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---

## `work_open`

Open a worked record: save an XRM record of `entity` (fields incl. any sub-type discriminator) and enroll it into its routed worklist queue. `contact_id` defaults to the conversation contact. Returns { record_id, record_number, queue_key } (queue_key null = unrouted). Ports: ok · failed.

### Inputs

| Field | Type | Required | Description |
|---|---|---|---|
| `entity` | string | ✓ |  |
| `fields` | object |  |  |
| `stage` | string |  |  |
| `contact_id` | string |  |  |

### Output ports

#### `ok`

| Field | Type | Required | Description |
|---|---|---|---|
| `record_id` | string | ✓ |  |
| `record_number` | number | ✓ |  |
| `queue_key` | union | ✓ |  |

#### `failed`

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | string | ✓ |  |

---
