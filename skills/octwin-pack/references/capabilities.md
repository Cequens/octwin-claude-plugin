# Capability reference — the platform KB (the source of truth)

The exhaustive "what can I use" reference is the **platform knowledge base** — pulled from your Octwin
platform, or the bundled snapshot that ships with this skill. Consult it instead of listing capabilities
from memory; if something isn't in the KB, it doesn't exist for a pure-YAML pack.

## Where it is (same layout in both)

- **Live:** `.octwin/platform-kb/` — after `octwin platform-kb pull` (matches your platform's exact version).
- **Bundled snapshot:** [`platform-kb/`](platform-kb/) — ships with this skill; the offline / first-run fallback.

Each holds **markdown docs** (read these first) + **JSON catalogs** (open for precise field schemas).

## What answers what

| File | Read it for |
|---|---|
| `dsl.md` | The flow YAML DSL — op-keys, expression language, output ports, top-level flow shape. |
| `flow-dsl-schema.md` | Canonical flow-runtime schema reference (prose) for authoring agents. |
| `flow-schema.json` | Machine schema for every node op-key + the full `render_intent` enum. |
| `declarations.json` | **JSON Schemas for the files you WRITE** to enable modules — `manifest.yaml`, `xrm.yaml`, `worklist.yaml` (casework), `scheduling.yaml`, plus surveys/automation/roles/journey/taps/commands/messages. The authoritative shape of each declaration. |
| `system-entities.json` | The platform's **built-in XRM entities** every pack gets free (`product`/`cart`/`order`/`case`/`booking`/…) — fields + pipelines. Compose flows over them, or `extends: system` in `xrm.yaml`. |
| `manifest-authoring.md` · `xrm-guide.md` · `scheduling-guide.md` · `commerce-guide.md` · `casework-guide.md` | Prose authoring guides for the manifest + first-class modules (the companion to the `declarations` / `system-entities` catalogs). |
| `primitives-guide.md` · `primitives.json` | Platform built-in `do:` primitives (`record_*`, `case_*`, `notify`, media, …) — readable guide + precise input/output-port JSON Schemas. |
| `builtins.json` | Expression-language builtins (`$coalesce`, `$map`, `$lines`, `$format_number`, …) usable in any `$expr`. |
| `templates-guide.md` · `templates.json` | Load-time `use:` subflow templates. |
| `channels-guide.md` · `channels.json` | Per-channel contact-metadata + inbound-ephemera field specs. |
| `runtime.md` | Flow kinds (tool / pre-turn / post-turn / media-handler) + the context namespaces they expose. |
| `taps.md` | Tap resolution — `taps.yaml` schema + structured `t:<verb>:<target>:<bindings>` ids. |
| `notify.md` · `media.md` | Cross-user delivery + the media pipeline (`MEDIA-` refs, inbound vision/voice). |
| `agent-protocol.md` | The universal agent contract (tool envelope, instructions-precedence, observational markers). |

## How to use it

1. **Choosing** a step / function / render-intent → skim the relevant `*.md`.
2. **Getting exact inputs/ports** for a primitive, or the fields of a render intent → open the matching `*.json`.
3. **Not in the KB → don't use it.** A pure-YAML pack can only use what the platform ships; there are no custom primitives.

The craft references here ([manifest.md](manifest.md), [flows.md](flows.md), [data-and-render.md](data-and-render.md))
teach *how to wield* these capabilities well; the KB is *what exists*.
