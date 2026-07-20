# 19 — Platform media service (`MEDIA-XXXXXXXX` refs + inbound pre-processing services)

**Status:** AUTHORITATIVE. Source files cited here are canonical truth.

**Pair with:** [12-pack-authoring.md](12-pack-authoring.md) (conventions) · [13a-dsl-reference.md](13a-dsl-reference.md) (step grammar) · [18-platform-notify.md](18-platform-notify.md) (sibling stdlib primitive).

---

## §1 The problem

Pre-Phase 11, every pack that handled inbound images / documents reinvented the same plumbing:

- Download bytes from the channel (WhatsApp media id resolution, web data-URL parsing).
- Upload to a bucket (each pack hardcoded its own bucket name + path scheme).
- Run domain analysis (vision / OCR).
- Stuff the resolved URL + analysis blob into the agent's context as a JSON payload.

Real-estate's `processImage` did all four steps in a 87-line TS hook. Documents went through a parallel 16-line hook. URLs (~100-120 tokens each) rode through the agent's LLM context — a 5-photo publish session leaked ~600-2000 tokens of URL strings into history. Storage details leaked into model context. And the only way to add media to a new pack was to re-implement the whole chain.

Phase 11 split responsibility cleanly: the **platform** owns storage + ref minting + URL plumbing. The Phase 13.5 *inbound pre-processing* module took the next step — analysis itself moved into platform-owned **services** (vision over images, transcription over voice) that a pack merely **enables** in its manifest (§6). The bespoke per-pack `media_handlers` YAML flow (formerly the only path) survives as the documented escape hatch for custom side-effects.

---

## §2 The contract

### `MEDIA-XXXXXXXX` reference scheme

Every inbound media asset gets a project-scoped, channel-agnostic ref derived from its UUID — the same 8-char hex-slice convention used elsewhere (`UID-`, `LID-`). Refs are:

- **Human-facing** — the agent reads them, logs render them, taps embed them.
- **Channel-agnostic** — same ref works whether the asset arrived via WhatsApp, web, voice, or any future channel.
- **Project-scoped** — resolution always filters by `currentProjectId()`; a ref minted in project A never resolves in project B.
- **Kind-distinct prefix** — voice notes (`kind === 'audio'`) mint `VOICENOTE-`; images/documents mint `MEDIA-`. The prefix is **cosmetic** (resolution is by the UUID hex slice, prefix-agnostic — both `isMediaRef` and `resolveMediaId` accept either), so it reads distinctly in the agent/XRM/console without changing lookup.

Source: [`src/platform/media/ref.ts`](../src/platform/media/ref.ts).

```typescript
mediaRef('550e8400-e29b-41d4-a716-446655440000')          // → 'MEDIA-E29B41D4'
mediaRef('550e8400-e29b-41d4-a716-446655440000', 'audio') // → 'VOICENOTE-E29B41D4'
```

### Agent protocol

The agent sees a minimal JSON envelope for every inbound media — no URL, no base64, no analysis blob:

```json
{ "type": "document",    "media_id": "MEDIA-EF56AB78", "mime_type": "application/pdf", "filename": "...", "caption": "..." }
{ "type": "media_group", "media": [ { "media_id": "MEDIA-AB12CD34", "kind": "image", "mime_type": "image/jpeg", "caption": "..." }, … ] }
```

A single document arrives as its own envelope; **images and voice notes coalesce** (§3) into ONE `media_group` whose items carry `kind: "image" | "audio"`. (A lone image is a degenerate one-item group.)

Tool inputs that accept media take **media_id refs**, NOT URLs. The renderer's last-mile expansion swaps refs for storage URLs before any channel adapter sees them. The universal contract lives in [`src/platform/agent/prompts/agent-protocol.md`](../src/platform/agent/prompts/agent-protocol.md) and is auto-appended to every pack agent.

---

## §3 The flow on every inbound media

Source: [`src/platform/channels/inbound/image.ts`](../src/platform/channels/inbound/image.ts) / [`audio.ts`](../src/platform/channels/inbound/audio.ts) / [`document.ts`](../src/platform/channels/inbound/document.ts) → shared [`_media-shared.ts`](../src/platform/channels/inbound/_media-shared.ts) → coalescing [`media-batch.ts`](../src/platform/channels/inbound/media-batch.ts).

```
channel webhook arrives (WA Cloud API / web upload)
  → resolveBuffer (download / decode data-URL)
  → uploadMedia({ buffer, mime_type, kind, caption, conversation_id })
       → StorageAdapter.upload(bucket, path, buffer)        bucket per kind (§4)
       → INSERT INTO media_assets (...) RETURNING *
       → returns { media_id, media_ref, storage_url, asset }
  → IMAGES + AUDIO: enqueueMediaBatch(...)  — buffer behind a short debounce window
       (MEDIA_BATCH_WINDOW_MS), then ONE coalesced turn:
         · group buffered items by kind
         · per kind → resolveInboundPreprocessing(kind) (§6):
              service → run it ONCE over the kind's items (vision: one analyzeImage
                        batch; voice: transcribe per item); append the combined note
              flow    → runMediaHandlerFlow(...) (the media_handlers escape hatch)
              none    → no analysis; media still reaches the agent
         · ONE agent turn whose user envelope is the `media_group` (§2)
  → DOCUMENTS: fire-now (no coalescing) — resolveInboundPreprocessing then
       handleTextMessage with the single-document envelope
```

A non-media inbound (text / tap / document) **flushes** any pending image/audio batch first, so ordering is preserved. The batch buffer is in-memory + single-instance (see [`docs/BACKLOG.md`](BACKLOG.md)).

PDFs get text-extracted platform-side (capped 8000 chars) and the result lands on `media_assets.metadata.extracted_text` before the document reaches the agent.

---

## §4 Storage adapter

[`src/platform/adapters/storage.ts`](../src/platform/adapters/storage.ts) defines the contract. Every method takes `projectId` first (routing is explicit, never ambient). The URL-minting `getUrl` was retired (2026-07-10) — the adapter only moves BYTES; URLs are minted by the media layer (`mediaServeUrl`, a relative `/api/media` URL) and serving streams via `download`.

```typescript
export interface StorageAdapter {
  upload(projectId, { bucket, path, buffer, mimeType, upsert? }): Promise<{ key: string }>  // key = logical path
  download?(projectId, bucket, key): Promise<Buffer>
  remove?(projectId, bucket, keys: string[]): Promise<void>                                  // batch
  list?(projectId, bucket, prefix?): Promise<StorageListEntry[]>
}
```

**Object storage is a per-project backend (migration 078)** — a `storage_kind` axis on `tenant_data_stores` (`supabase` | `local` | `s3`), ORTHOGONAL to the DB kind. The registered adapter is a **`StorageDispatcher`** ([`storage.dispatch.ts`](../src/platform/adapters/storage.dispatch.ts)) that resolves the project's `storage_kind` per call and delegates to a driver:

- **`supabase`** ([`storage.supabase.ts`](../src/platform/adapters/storage.supabase.ts), default) — reuses the per-project `SupabaseClient` from `DataStoreAdapter` (the install-bucket-wrapped client; DB + storage share it). A `bundled`/`postgres` project falls back to the platform BUNDLED substrate (path-scoped by `<project_id>/`). No new credentials.
- **`local`** ([`storage.local.ts`](../src/platform/adapters/storage.local.ts)) — server disk, `<STORAGE_LOCAL_ROOT>/<projectId>/<bucket>/<key>`.
- **`s3`** ([`storage.s3.ts`](../src/platform/adapters/storage.s3.ts)) — AWS S3, or MinIO/R2 via `S3_ENDPOINT` + `S3_FORCE_PATH_STYLE`.

The logical `key` (= `input.path`) is stored in `media_assets` and stays portable across drivers; the driver namespaces physically by `projectId`. Buckets are kind-scoped: `media-images` / `media-documents` / `media-audio`. Path scheme: `<project_id>/<conversation_id>/<timestamp>.<ext>`.

Set the platform default with `STORAGE_DRIVER`; override per project via the data-store config (console) or SQL. Move existing bytes between backends with `scripts/migrate-storage-driver.ts` (preserves `(bucket, key)` so `media_assets` rows resolve unchanged). Backups + pack `db.storage` follow the same per-project backend.

---

## §5 The `media_assets` table

[`src/platform/db/migrations/017_media_assets.sql`](../src/platform/db/migrations/017_media_assets.sql). Lives in the PLATFORM database alongside `contacts` / `conversations` (not the tenant's data store — that holds domain blobs).

Key columns:

| Column | Purpose |
|---|---|
| `id` UUID | Source of the `MEDIA-XXXXXXXX` ref (chars 8..16). |
| `project_id`, `tenant_id` | Project scoping; ref lookups filter by project. |
| `uploader_contact_id`, `conversation_id` | Audit + admin timeline linkage. |
| `case_id`, `record_id` | Domain-module linkage ([`040_media_case_record_links.sql`](../src/platform/db/migrations/040_media_case_record_links.sql)) — an asset attached to a casework case / XRM record. Both nullable, `ON DELETE SET NULL` (assets outlive a deleted case/record). Written by `case_attach` / a `record_save` media-field ref; read at the route tier for the console (§8b). |
| `kind` | `'image'`, `'document'`, or `'audio'` (DB check constraint — [`033_media_assets_audio_kind.sql`](../src/platform/db/migrations/033_media_assets_audio_kind.sql)). |
| `mime_type`, `byte_size` | Raw media metadata. |
| `storage_bucket`, `storage_key` | Logical location; passed to `storage.download(projectId, bucket, key)` to stream bytes (driver-portable across supabase/local/s3). |
| `caption` | User-supplied caption from the channel. |
| `metadata` JSONB | Analysis blobs land here — the inbound pre-processing service writes `metadata.analysis` generically (§6); a `media_handlers` flow may write its own shape. |

Indexes: `(project_id, created_at DESC)`, `(conversation_id, created_at DESC)`, and a functional index on the middle-hex slice for `MEDIA-` ref resolution.

---

## §6 Pack-side: enabling inbound pre-processing (the primary path)

A pack **configures a built-in service per inbound kind** in its manifest — no flow, no note formatting. Module + service interface live in [`src/platform/media/preprocessing/`](../src/platform/media/preprocessing/README.md); the schemas are `preprocKindSchema` (image/document) and `voicenotePreprocSchema` (voice notes) in [`src/platform/manifest/schema.ts`](../src/platform/manifest/schema.ts).

```yaml
# src/packs/<pack-id>/manifest.yaml
inbound_preprocessing:
  image:                              # image/document → vision service, toggled by `enabled`
    enabled: true
    ack: '📷 ...'                     # optional: status text sent ONCE when the batch starts (channel layer sends it)
    prompt: |-                        # optional: the domain analysis prompt (vision JSON shape)
      ...respond with JSON {"images":[{"media_ref":..., "image_type":..., ...}]}...
    note_template: '$lines([...])'    # optional: per-item note block (see below)
  voicenote:                          # the `audio` kind — a `mode` enum, NOT `enabled`
    mode: transcribe                  # transcribe | passthrough | decline
    ack: '🎤 ...'                     # transcribe: status text while STT runs
    prompt: 'Egyptian Arabic ...'     # transcribe: STT language/hint
    note_template: '...'              # transcribe: per-item briefing block
    # notice: '...'                   # passthrough (optional) / decline (required): user-facing message
```

| field | meaning |
|---|---|
| `enabled` (image/document) | turns the kind's built-in vision service on. |
| `mode` (voicenote) | `transcribe` = STT + agent turn; `passthrough` = NO STT, ref-preserving floor note so the agent/collect-engine can still use the ref; `decline` = NO STT, NO agent turn, send `notice`. An **absent** `voicenote` block defaults to `passthrough`. |
| `prompt` | domain analysis prompt. Vision: instruct a JSON shape and echo each `media_ref` in order. Voicenote (`transcribe`): a free-text STT **biasing** hint (domain/dialect context, e.g. "Saudi Arabic ride-hailing voice note") — Whisper's `prompt`, NOT a language code; language auto-detects. Omitted → platform default. |
| `note_template` | a flow-expression rendered **per analyzed item** to build that item's briefing block (`{$field}` interpolation, `$field` eval, `$lines`/`$length` helpers — same grammar as flow `args:`). Omitted → a generic `key: value` dump of the analysis JSON. |
| `ack` | one status message sent when the batch begins; the **channel layer** sends it (the service has no channel access). |
| `notice` (voicenote) | user-facing message on the `passthrough` (optional) / `decline` (required) paths. |

The service does the rest: resolves URLs → ONE inference call per kind (vision batches N images into a single `analyzeImage`; voice transcribes per item) → persists `media_assets.metadata.analysis` generically → builds the ONE combined note (a fixed **English** usage instruction + a per-item block) → appends it to the turn store, drained by `PreTurnBriefingProcessor` as `<system-reminder type="pre_turn_briefing" source="flow">`. Real-estate's config is the worked example: [`src/packs/real-estate/manifest.yaml`](../src/packs/real-estate/manifest.yaml) (`inbound_preprocessing:`).

**Documents:** `inbound_preprocessing.document` routes through the same vision service as `image` (`SERVICE_BY_KIND` in [`registry.ts`](../src/platform/media/preprocessing/registry.ts)) and fires immediately (1-element batch, not coalesced). PDFs additionally get up-front text extraction (§3) stashed on `metadata.extracted_text`, so a `document` prompt can lean on either signal.

**Pack-agnostic invariant:** the module holds zero domain strings — the prompt, JSON shape, and note labels come only from the pack's manifest; the persist target is the generic `media_assets.metadata`.

---

## §7 Escape hatch: `media_handlers:` custom flow

When a kind needs custom side-effects the built-in service can't express, a pack declares a YAML flow instead. The manifest **forbids enabling both** `inbound_preprocessing` (enabled) and `media_handlers` for the same kind (`superRefine` in the schema).

```yaml
# src/packs/<pack-id>/manifest.yaml
media_handlers:
  image:    my-media-flow       # flow id (must be declared in flows:)
  document: my-media-flow
```

The flow runs after upload via `runMediaHandlerFlow(...)` — DIRECTLY (no user resolution, no LLM loop) — and returns `memory_notes` through its result envelope (the native `note:` node — formerly `emit_envelope.memory_note`; the legacy `do: inject_memory_note` primitive was retired in Phase 15). `PreTurnBriefingProcessor` drains them like the service's note. The flow receives `{ media_id, media_ref, mime_type, caption, kind }` and resolves bytes with `do: get_media` (§8). Showcase keeps a trivial `ack-media` flow as the live example. The flow must be `internal: true` (§7a).

### §7a `internal: true` — platform-only flow visibility

A `media_handlers` flow is invoked by the platform, never by the agent. The `internal: true` flag enforces that:

- **Excluded from the Mastra `tools:` map** — the agent's tool resolver doesn't see internal flows.
- **Boot-time validator** in [`buildPackTools`](../src/platform/pack/tools.ts) throws if any `agents.<id>.tools:` list erroneously includes an internal flow id.
- **Still constructed + registered** — the FlowTool lives in the per-pack registry (`PackRegistry.getFlowRegistry(packId).get(flowId)`) so platform code (`runMediaHandlerFlow`, future cron / webhook / on-publish triggers) can invoke it.

Use `internal: true` for:
- Media-handler escape-hatch flows (this doc).
- Cron / scheduled jobs (future).
- Webhook receivers (future).
- Sub-flows that exist only for `use:` / `goto:` composition from another flow.

Tested in [`internal-flow.test.ts`](../src/platform/pack/internal-flow.test.ts).

---

## §8 Reading + writing media from pack primitives

Pack TS handlers never import the platform media internals (`uploadMedia`, `resolveMediaUrl`, `getMediaById`). Instead, pack primitives accept resolved values (`url`, `mime_type`, etc.) and the YAML upstream uses `do: get_media` to resolve from a `media_id`:

```yaml
# Pack flow that needs the actual bytes of a referenced image
- do: get_media
  args: { media_id: '$input.image_media_id' }
  bind: m
  outputs:
    found:
      - do: my_pack_primitive
        args:
          url:       '$m.url'
          mime_type: '$m.mime_type'
          extracted: '$m.metadata.analysis.extracted_text'
    not_found: []
```

This keeps the platform's `media_assets` table + storage adapter behind a stable flow-primitive boundary — the platform can swap Supabase Storage for S3 without touching any pack code.

---

## §8a Requesting an upload mid-flow: `collect: media`

A flow asks for a photo/document deterministically with a `type: media` collect field (DSL reference — the `collect` section). The engine synthesizes a 📷/📎 gap, suspends, and on the next turn **folds the uploaded `MEDIA-` ref into `$state.<field>`** from a per-turn `$inbound_media` list — independent of how the agent re-invokes (a wrong-kind upload re-prompts).

`$inbound_media` (`[{ ref, kind }]`) is set by the channel media pipeline around the agent turn via the kernel [`withInboundMedia`](../src/platform/kernel/tenant-context.ts) ALS: [`runCoalescedTurn`](../src/platform/channels/inbound/media-batch.ts) for the coalesced image/audio batch and the document fire-now path in [`_media-shared.ts`](../src/platform/channels/inbound/_media-shared.ts). [`FlowTool`](../src/platform/flow-runtime/flow-tool.ts) reads it via `currentInboundMedia()` and passes it on both the fresh-run and resume legs; `resumeFlow` binds it **fresh per turn** (never restored from the suspend snapshot). The engine's adopt/wrong-kind/gap logic lives in `executeCollectNode` ([`node-handlers.ts`](../src/platform/flow-runtime/interpreter/node-handlers.ts)). Routing stays on the existing agent-turn path (the channel layer sits below flow-runtime — it can't call `runWorkflowResume` directly); only the *fold* is made deterministic.

## §8b Reflecting media on domain records + cases

Uploads surface on the operator console beyond the conversation viewer / Media page:

- **XRM records** — a field declared `image`/`document` in `xrm.yaml` stores a `MEDIA-` ref. Existence + kind are checked against `media_assets` by inline SQL in [`verifyFieldRefs`](../src/platform/xrm/services/record-service.ts) (xrm is below `media` in the layer-DAG, so it must not import media). The record-detail route ([`src/routes/xrm.ts`](../src/routes/xrm.ts)) resolves the refs to signed URLs via [`resolveMediaViews`](../src/platform/media/url.ts) and the page renders a thumbnail/link.
- **Casework cases** — the [`case_attach`](../src/platform/primitives/casework/case_attach/primitive.ts) primitive (rank 24, above both media + casework) writes the queryable link (`media_assets.case_id`, via the media store's `linkMediaToCase`) AND an `attachment` timeline event (casework's `appendAttachment`) — neither module imports the other. The case-detail route lists a case's attachments (`listCaseAttachments` + signed URLs) into an **Attachments** card.

---

## §9 Renderer last-mile: `image_url:` accepts MEDIA-refs

The flow-render pipeline accepts MEDIA-refs in any URL-bearing field. Two integration points:

- [`ChannelBridge.renderHint`](../src/platform/channels/bridge.ts) walks the resolved UiHint via [`expandMediaRefs`](../src/platform/media/expand-refs.ts) and rewrites refs in-place before dispatching to the channel impl.
- [`WhatsAppMessagingAdapter.sendIntent`](../src/platform/channels/whatsapp/messaging.ts) does the same after `renderIntent` + `compileOnSelectsInTree`.

Fields scanned (allowlisted by key name): `image_url`, `url`, `image`, `header_image_url`, `thumbnail_url`. Refs that fail to resolve are logged + left as-is; the renderer's existing `caps.image.is_renderable` check downstream substitutes the intent's `placeholder_url`.

Cards with mixed values (some URLs, some refs) work fine — only fields matching `MEDIA-XXXXXXXX` are rewritten.

---

## §10 What replaced what

| Before (Phase 10 and earlier) | After (Phase 11 → 13.5) |
|---|---|
| `PackChannelHooks.processImage` / `processDocument` (TS hook) | `manifest.inbound_preprocessing.<kind>` → built-in platform service (§6); `media_handlers` flow is the escape hatch (§7) |
| per-pack `analyze-media` vision flow (~90 lines YAML + primitives) | `inbound_preprocessing.image.{prompt,note_template}` — config only, no flow |
| image-only inbound; one agent turn per photo | images + voice notes coalesce into ONE `media_group` turn (§3) |
| Pack `storage.service.ts` (per-pack Supabase upload) | Platform `StorageAdapter` slot + `uploadMedia` |
| Full URL in agent context (~100-120 tokens / asset) | `MEDIA-XXXXXXXX` (~25 tokens / asset) |
| `photo_analyses: [{ url, analysis }]` field on `publish-listing` | `media_assets.metadata.analysis` (off-thread) |
| `image_url` + `image_base64` + `image_mime_type` on `lead-action` | Single `image_media_id` ref |

Footprint comparison for a 5-photo publish session (typical real-estate use):

- **Before:** ~600-2000 tokens of URL strings + `analysis` blobs in agent context.
- **After:** ~125 tokens (5 refs + caption + minimal type marker).

---

## §11 Out of scope (deferred)

- Lifecycle / cleanup policy (orphaned uploads from cancelled flows). Future cron.
- Web voice notes — the web widget records via the browser `MediaRecorder` and uploads `type: 'audio'` multipart, same path as WhatsApp voice notes (the recorded Opus-in-WebM/Ogg or AAC-in-MP4 is transcribed by the same `voice-transcription` service). The transcript is pushed back to the widget via a `transcript` SSE envelope (keyed by the bubble's `local_id`) and shown under the voice-note bubble, WhatsApp-style.
- Backfilling existing `properties.photos` URL arrays to MEDIA-refs (renderer accepts both, so no urgency).
- Cross-project media sharing (today: same-project scoping enforced at lookup).
- Multi-GB streaming uploads (today: in-memory buffer).
- Signed-URL TTL refresh caching (renderer pre-resolves every render, so signed URLs stay fresh).
