# Platform Stdlib — Channels: `$caps` + Context

> ⚙️ **Auto-generated** from the per-channel `ChannelCaps` exports (`waChannelCaps` / `webChannelCaps`) + `src/platform/channels/<channel>/<channel>.context.ts`. Do NOT hand-edit — rerun `npm run platform-kb:channels` to regenerate. The caps shown are the SAME objects the runtime binds as `$caps`, so these numbers cannot drift from what flows see.

`$caps` is the platform-injected channel-limits object flows read (`$caps.list.maxRows`, `$caps.buttons.max`, `$caps.carousel.max`, `$caps.carousel.buttonsPerCard`). Primitives + YAML stay channel-blind — the renderer clamps — but flow-control decisions (e.g. "skip the picker when results fit one render") legitimately read these values.

Every channel adapter also declares its **per-contact durable attributes** (persisted to `Contact.metadata.<channel>.<field>`) and **per-message ephemera** (exposed to flows as `$inbound.<channel>.<field>` once Phase C ships). The platform inbound boundary write-through reads these specs implicitly via the runtime metadata-namespace validator — extending them is a pure additive change.

See `Contact.metadata` doc in [`src/platform/db/types.ts`](../src/platform/db/types.ts) for the two-namespace convention overview.

2 channels declared:

| Channel | List rows | Buttons | Carousel cards | Buttons/card | Durable attributes | Per-message ephemera |
|---|---|---|---|---|---|---|
| [`whatsapp`](#whatsapp) | 10 | 3 | 10 | 2 | 0 fields | 0 fields |
| [`web`](#web) | 10 | 3 | 10 | 2 | 0 fields | 0 fields |

Companion catalogs:
- Machine-readable JSON: [`dist/octwin-platform-kb/channels.json`](../dist/octwin-platform-kb/channels.json)
- Platform builtin primitives: [`16b-platform-stdlib-primitives.md`](./16b-platform-stdlib-primitives.md)

---

## `whatsapp`

### `$caps` — hard limits (verbatim)

```json
{
  "list": {
    "maxRows": 10
  },
  "buttons": {
    "max": 3
  },
  "carousel": {
    "max": 10,
    "buttonsPerCard": 2
  },
  "text": {
    "headerMax": 60,
    "footerMax": 60,
    "bodyMax": 1024,
    "rowTitleMax": 24,
    "rowDescMax": 72,
    "buttonTitleMax": 20,
    "carouselBodyMax": 160
  },
  "commerce": {
    "native_catalog": true,
    "native_cart": true,
    "supports_payments": true,
    "products_per_message_max": 30,
    "sections_max": 10,
    "section_title_max": 24,
    "images_per_item_max": 10
  },
  "image_renderable": "URL path must end in .jpg/.jpeg/.png (Meta silently drops other links); failures get the placeholder_url substituted"
}
```

### `Contact.metadata.whatsapp.*` — durable per-contact attributes

_No durable attributes declared today._ The adapter writes nothing channel-specific to `Contact.metadata`. Scaffolding is in place; extending is a pure additive change (declare the field here + emit it from the adapter's parser).

### `$inbound.whatsapp.*` — per-message ephemera

_Phase C namespace — declared today, exposed to flows once the runtime pipeline lands._

_No per-message ephemera declared._

---

## `web`

### `$caps` — hard limits (verbatim)

```json
{
  "list": {
    "maxRows": 10
  },
  "buttons": {
    "max": 3
  },
  "carousel": {
    "max": 10,
    "buttonsPerCard": 2
  },
  "text": {
    "headerMax": 60,
    "footerMax": 60,
    "bodyMax": 4096,
    "rowTitleMax": 256,
    "rowDescMax": 1024,
    "buttonTitleMax": 256
  },
  "commerce": {
    "native_catalog": false,
    "native_cart": false,
    "supports_payments": true,
    "products_per_message_max": 30,
    "sections_max": 10,
    "section_title_max": 24,
    "images_per_item_max": 10
  },
  "image_renderable": "any non-empty URL renders"
}
```

### `Contact.metadata.web.*` — durable per-contact attributes

_No durable attributes declared today._ The adapter writes nothing channel-specific to `Contact.metadata`. Scaffolding is in place; extending is a pure additive change (declare the field here + emit it from the adapter's parser).

### `$inbound.web.*` — per-message ephemera

_Phase C namespace — declared today, exposed to flows once the runtime pipeline lands._

_No per-message ephemera declared._

---
