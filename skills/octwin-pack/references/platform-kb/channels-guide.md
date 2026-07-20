# Platform Stdlib — Channel Context

> ⚙️ **Auto-generated** from `src/platform/channels/<channel>/<channel>.context.ts`. Do NOT hand-edit — rerun `npm run platform-kb:channels` to regenerate. Source of truth is the TypeScript `ChannelContextSpec` exports; this page mirrors them for human consumption.

Every channel adapter declares its **per-contact durable attributes** (persisted to `Contact.metadata.<channel>.<field>`) and **per-message ephemera** (exposed to flows as `$inbound.<channel>.<field>` once Phase C ships). The platform inbound boundary write-through reads these specs implicitly via the runtime metadata-namespace validator — extending them is a pure additive change.

See `Contact.metadata` doc in [`src/platform/db/types.ts`](../src/platform/db/types.ts) for the two-namespace convention overview.

0 channels declared:

| Channel | Durable attributes | Per-message ephemera |
|---|---|---|

Companion catalogs:
- Machine-readable JSON: [`dist/octwin-platform-kb/channels.json`](../dist/octwin-platform-kb/channels.json)
- Platform builtin primitives: [`16b-platform-stdlib-primitives.md`](./16b-platform-stdlib-primitives.md)

---
