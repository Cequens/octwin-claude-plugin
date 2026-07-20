# 23 — Commerce (Meta WhatsApp commerce + web simulation)

> 📘 **Module guide Artifact:** <https://claude.ai/code/artifact/e6488789-704c-4c1a-9ba0-b690cc05674b>
> (source [`docs/artifacts/commerce-module-guide.html`](artifacts/commerce-module-guide.html)).

The platform ships a generic, **pack-agnostic commerce capability**: a product catalog,
Single/Multi-Product messages, the "View catalog" CTA, a cart→order round-trip, and checkout
(`order_details`/`order_status`). It renders natively on WhatsApp (Meta commerce) and is
**simulated byte-faithfully on the web widget**. The shipped demo is the `ecommerce` pack.

The product schema is Meta's universal Catalog Fields spec — a **channel primitive, not domain
data** — so it lives platform-side exactly like `contacts`/`conversations`. In the catalog
convergence the catalog became the **`product` XRM system entity** (searched by the generic
`record_search` / `record_related`), owned by the [`catalog`](../src/platform/catalog/README.md)
module above xrm — the retired `commerce.products` table + `search_products`/`recommend_related`
pgvector RPCs are gone. Packs compose the commerce stdlib in pure YAML; `real-estate` is unaffected
(properties aren't Meta SKUs).

## Where things live

The [`commerce`](../src/platform/commerce/) layer (rank: after `provisioning`, before
`flow-runtime` — see [`scripts/layers.mjs`](../scripts/layers.mjs)) owns the domain core; the
layer-bound surfaces stay thin in their host layers and delegate into it (the platform discovers
render intents from `flow-runtime/render-intents`, primitives from `primitives/`, senders from
`channels/`).

- **`commerce/contracts/money.ts`** — `Money { value, offset }` (real = value/offset). One helper
  converts wire ↔ display ↔ catalog **minor-units** ↔ feed string. The 100× footgun lives nowhere
  else.
- **`catalog/catalog-service.ts`** ([`src/platform/catalog/`](../src/platform/catalog/README.md)) —
  the catalog domain seam over the **`product` xrm system entity**: `upsertProducts` (dedupe by
  `retailer_id`, baked embeddings honoured), `getProduct`, `listProducts`, `searchProducts`
  (→ `record_search`), `recommendRelated` (→ `record_related`), `setAvailability`, `deleteProduct`
  (soft-archive), and `pruneMetaProductsNotIn` (a pull is a *sync*, not an append).
- **`catalog/meta-sync.ts`** — `items_batch` push + paginated **pull** + `readReviewStatus`
  (`parsePrice` strips thousands separators). Reads/writes product **records**; the wire contract is
  byte-identical to the retired commerce meta-sync.
- **`commerce/catalog/meta-onboarding.ts`** — catalog **discovery** via `owned_product_catalogs` +
  the WABA edge, Graph `product_catalogs` connect, `whatsapp_commerce_settings`, and the
  `getCommerceReadiness` probe (Graph-only; stays in `commerce`).
- **`cart/cart-service.ts`** ([`src/platform/cart/`](../src/platform/cart/README.md)) — the cart seam
  over the **`cart` xrm system entity** (platform DB). `stage` = cart status (`CART_STATUS_TRANSITIONS`:
  `open → submitted / abandoned`); lines live in the E1 `line_items` field and the engine rolls up the
  `total`. **Decoupled from the catalog**: a line snapshots name/price/image at add-time (the `cart_add`
  primitive resolves them from `#platform/catalog`), so the cart never reads product records. Flows
  render **precomputed** totals; line edits write `silent` (no per-tweak timeline event).
- **`orders/order-service.ts`** ([`src/platform/orders/`](../src/platform/orders/README.md)) — the
  order domain seam. An order is a record of the **`order` xrm system entity** (platform DB):
  `stage` = fulfillment status (strict `ORDER_STATUS_TRANSITIONS` pipeline), `payment_status` /
  money / `items` are fields.
- **The `commerce` schema is now empty.** The former `commerce.carts` / `cart_items` tables were
  **retired** with the cart's move to XRM — migration
  [`061_commerce.sql`](../src/platform/db/migrations/061_commerce.sql) now only creates the (empty)
  schema and [`069_drop_commerce_cart_tables.sql`](../src/platform/db/migrations/069_drop_commerce_cart_tables.sql)
  drops the orphan tables on legacy dev DBs. (The retired `commerce.products` catalog + its RPCs went in
  [`064`](../src/platform/db/migrations/064_drop_retired_tables.sql).)

## Storage topology

Everything commerce persists lives on the **control-plane platform DB** — the same tier as
`contacts`/`conversations`/`xrm`. Products are **`product` XRM records** (`xrm_records`, searched by
the pgvector+pg_trgm `record_search`); the **cart** is a **`cart` XRM record** (lines in the
`line_items` field, engine-rolled `total`); **orders** are `order` XRM records. All are
`project_id`-scoped (app-layer isolation, the XRM/casework model). So a commerce pack owns **no**
`data-store`: `ecommerce` declares `[messaging, embedding,
storage]` and installs with no DB prompt. It is **channel-independent**: a web-only project gets a
full catalog→browse→cart→order→checkout with **zero Meta dependency**. Meta is an optional *sync
target* (the `catalog_id` binding) only relevant when WhatsApp is present.

> Historical note: commerce formerly lived on each project's served DB (the only reason was
> pgvector), then moved to a platform-owned `commerce` schema; the product catalog became XRM records
> (`commerce.products` + its `search_products`/`recommend_related` RPCs retired), orders became `order`
> records, and finally the cart became a `cart` record (`commerce.carts`/`cart_items` retired) — the
> `commerce` schema is now empty and commerce owns no records.

## The stdlib (what packs compose)

- **Primitives** ([`primitives/commerce/`](../src/platform/primitives/commerce/)):
  `product_search` (hybrid fuzzy+semantic — a thin wrapper over the generic `record_search`),
  `catalog_get`, `catalog_list`, `recommend_related` (→ `record_related`);
  `cart_add`/`cart_view`/`cart_update_qty`/`cart_remove`/`cart_clear`/`cart_submit`;
  `order_get`/`order_update_status`; `payment_request`. (The catalog reads can equally be the raw
  XRM `record_search`/`record_get`/`record_related` on the `product` entity — the commerce
  primitives just pre-shape render rows + map major-unit price filters.)
- **Render intents** (channel-blind): `product_card` (SPM), `product_list` (MPM), `catalog_link`,
  `order_details`, `order_status`. The WA renderer reduces products to `(catalog_id, retailer_id)`
  (resolved from the binding); the widget draws the rich card from carried data, degrading to text
  when no Meta catalog is bound. **Checkout** (`order_details`/`order_status`) degrades on WhatsApp:
  the native Meta Pay `review_and_pay`/`review_order` cards need WhatsApp Payments onboarding
  (region-gated — Meta `#131009` on a standard WABA), so the WA renderer emits a **CTA-URL to the
  hosted pay page** (`payment_url` present) or **plain text** (pay-on-delivery, no `payment_url`).
  The widget still renders the byte-faithful Meta checkout card. Native Meta Pay is backlogged
  ([`BACKLOG-meta.md`](./BACKLOG-meta.md)).
- **Builtin**: `$sum`. **Caps**: `ChannelCaps.commerce`.

## Cart → order convergence

WhatsApp's native cart submits an inbound `order` ([`channels/inbound/order.ts`](../src/platform/channels/inbound/order.ts));
the web widget drives the server-side cart via taps and `cart_submit`. Both create an
`order`-entity record keyed by `reference_id` (the join key for `order_details` ↔ the payment
webhook ↔ `order_status`) and continue to the checkout flow. **The shopper never sees
`reference_id`** — the customer-facing order number is the record's `record_number` (rendered
`#1042`, exposed on `OrderView.record_number`); `reference_id` (the opaque `ord_…`) stays internal
plumbing. `cart_submit` is a two-record
composition (read the `cart` record's lines → create the `order` record → transition the cart
`open → submitted`; no shared transaction — the retry-dedupe fix is backlogged). A Meta payment status arrives as
`statuses[type=payment]` and patches the record's `payment_status` field (advisory — confirm via
Payment Lookup before fulfilling); the patch lands on the record timeline as a `field_change`
event, an audit trail the retired SQL `UPDATE` never had.

## Operator surface — the Commerce section

The console sidebar's **Commerce** section holds two route-backed tabs — **Orders** (the work-queue:
filter by status/payment, open an order for its item snapshot, totals, and transition buttons rendered
from `ORDER_STATUS_TRANSITIONS`) and **Catalog** (product editor + Meta catalog sync). The whole section
only appears when the project's pack actually does commerce (has a product, an order, or a bound Meta
catalog — the `commerce` key of [`GET .../capabilities`](../src/routes/capabilities.ts)) **and** the
tenant's plan bought `orders`/`catalog`.

The commerce **KPI tiles** (orders by status, pending payments, catalog size + binding) formerly had
their own "Overview" row here; they now live alongside the customer-operations tiles on the single
project **Overview** dashboard (the retired separate `CommerceOverviewPage`/`CustomerOsOverviewPage` were
merged into `ProjectOverviewPage`). A contact's orders also appear as an **Orders card** on the
contact-detail page (they arrive with the generic records read — an order IS a record).

## Seeding & console

A pack declares `commerce_catalog_seed: { data, assets_dir }` in its manifest. `seed:gen <pack>`
AI-generates product images **and** writes `catalog.seed.json` with **baked embeddings** (no model
call at provision). The seed applies at provision (the injected `applyCommerceSeed` hook) and from
the console **"Seed demo data"** action via [`src/commerce-catalog-seed.ts`](../src/commerce-catalog-seed.ts).
The console **Catalog** page (`/tenants/:slug/projects/:projectSlug/catalog`) is the canonical
editor/browser + Meta-catalog connect (the platform-owned catalog doesn't appear on the Database
page, which lists pack install schemas).

## Operator onboarding, sync & readiness (console Catalog page)

The binding card drives the whole Meta round-trip; all routes are under
`/api/admin/.../catalog` ([`src/routes/catalog.ts`](../src/routes/catalog.ts)):

- **Discover** (`GET …/catalogs?waba_id=`) — `discoverMetaCatalogs` queries **both**
  `owned_product_catalogs` (Business) and `product_catalogs` (WABA), so pasting a **WABA *or*
  Business id** works (the WABA edge alone is empty before a catalog is connected, and the bare
  `product_catalogs` edge on a User/Business returns Graph error **#12**). Returns `{ id, name }` so
  the operator picks by name and never pastes a catalog *name* as an id.
- **Connect** (`POST …/binding`) — persists `catalog_id` onto the WhatsApp webhook `config_json`
  (the load-bearing step) + best-effort Graph link. The Meta channel config also carries an optional
  **WABA id** field ([`channels/contracts/specs.ts`](../src/platform/channels/contracts/specs.ts)) —
  set it once and discovery/readiness prefill from it.
- **Import from Meta** (`POST …/pull`) — Meta-as-source. Pulls the bound catalog into
  `commerce.products` (`source='meta'`, baked-in embeddings), tags `metadata.meta_catalog_id`/`name`
  for provenance, then **prunes** meta-sourced rows not in that catalog so rebinding cleanly swaps
  the local set. Native/seeded rows are never pruned.
- **Push to Meta** (`POST …/sync`) — author-side → Meta (`items_batch`). Don't push into a catalog
  Meta already owns.
- **Check WhatsApp readiness** (`GET …/readiness`) — `getCommerceReadiness` renders a live checklist
  (catalog↔WABA, cart/visibility, business verification, per-product review) with Meta's own
  `possible_solution` per failing item; it also mirrors review state via `readReviewStatus` so the
  table's STATUS badges reflect reality. **STATUS is Meta-only** — web-only projects (no binding)
  show "web-only" instead of leaking the `pending` DB defaults.

## Gotchas (learned against a live WABA)

- **`#131009 product not found … in catalog_id`** on an SPM/MPM send, with the product clearly
  present in the catalog, means the item is **not yet in the WhatsApp messaging index** — i.e. its
  `capability_to_review_status[WHATSAPP]` is not `APPROVED`. Fresh items sit at `NO_REVIEW` until
  Meta approves them (up to ~24h); approved items send.
- **Use `capability_to_review_status[WHATSAPP]`, not the top-level `review_status`.** The latter
  comes back **empty** even for items Commerce Manager shows as "Product is shown"; only the
  capability value is authoritative. `readReviewStatus` mirrors the capability into
  `commerce.products.review_status` (`APPROVED`→`approved`, else `pending`).
- **Catalog vertical does NOT gate WhatsApp sends.** An `articles_and_publications` catalog and a
  `commerce` catalog behave identically — both gate on per-item review, not on vertical.
- **Business verification is a volume cap, not a send blocker.** `business_verification_status:
  not_verified` shows as `can_send_message: LIMITED` (Graph error `141010`) but approved products
  still send; readiness flags it as a warning, not a failure.
- **Send-guard (deferred):** the WhatsApp product-send path does **not** yet skip non-`approved`
  retailer_ids — an unapproved item `#131009`s at runtime. Now that `readReviewStatus` populates
  `review_status`, the guard is buildable — tracked in [`BACKLOG.md`](./BACKLOG.md).
- **`#131009 "Unsupported Interactive Message type"`** on an `order_details`/`order_status` send
  means the WABA isn't onboarded to **WhatsApp Payments** (Meta Pay is region-gated: IN/BR/SG).
  These interactive types are unavailable on a standard WABA, so the WA renderer no longer sends
  them — it degrades to a CTA-URL / plain-text order summary (see *Render intents* above). Native
  Meta Pay is backlogged in [`BACKLOG-meta.md`](./BACKLOG-meta.md).

## Wire reference + region notes

Byte-exact payloads + the region matrix (native Pay = India/Singapore; `payment_link` is the
globally-shippable rail v1 ships) are in the approved plan. Deferred items are in
[`BACKLOG.md`](./BACKLOG.md) (native WhatsApp Pay certification, multi-market catalogs, the
WhatsApp-send review-status gate, external commerce-source adapters).

For the broader **WhatsApp/Meta Graph API surface** beyond commerce — WABA provisioning, message
templates, messaging health/capping, Flows, profile, analytics, calling — and the current-vs-Meta gap
map, see the dedicated [`BACKLOG-meta.md`](./BACKLOG-meta.md).
