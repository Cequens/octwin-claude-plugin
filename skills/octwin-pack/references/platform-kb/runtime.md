# Platform Runtime — Flow Types & Context

> 📝 **Hand-curated.** Edit this file directly when adding a new flow kind or a new context namespace. The Reference page in the operator console renders it verbatim — changes are picked up at the next request (60s cache).

The platform runs **four kinds of flow** and exposes a fixed set of **context namespaces** that flow expressions can read. Both lists are stable platform contracts — new entries land via runtime changes, not pack edits. This page is the canonical reference.

Companion catalogs:
- [Primitives](./16b-platform-stdlib-primitives.md) — the verbs YAML can call via `do:`
- [Channels](./16c-platform-stdlib-channels.md) — per-channel `Contact.metadata.<channel>.*` + `$inbound.<channel>.*` fields
- [Templates](./16-platform-stdlib-templates.md) — load-time `use:` expansion
- [Partials](./16a-platform-stdlib-partials.md) — render-time `$partial()` expansion

---

## Flow types

Every flow has a `flow_kind` that controls how, when, and with what bindings it runs. The `buildFlowRunContext()` helper at [src/platform/flow-runtime/flow-run-context.ts](../src/platform/flow-runtime/flow-run-context.ts) is the single chokepoint — every invocation site passes the kind through it so context shape stays consistent across kinds.

| Kind | Trigger | Caller | Bindings | Cached? |
|---|---|---|---|---|
| `tool` | LLM tool call | Mastra agent shell | `$conversation`, `$contact`, `$input`, `$pre_turn`, `$inbound`, `$config`, `$vars` | No |
| `pre_turn` | Every inbound, before the agent runs | `runPreTurnFlows` | `$conversation`, `$contact`, `$input`, `$inbound`, `$config`, `$vars` (no `$pre_turn` — it's the accumulator being built) | Yes — per `(tenant, channel, channel_contact_handle, flow_id)`, TTL declared in flow's `cache_ttl_s` |
| `post_turn` | After the agent emits a reply | `runPostTurnFlows` | `$conversation`, `$contact`, `$input`, `$pre_turn`, `$inbound`, `$config`, `$vars` | No |
| `media_handler` | Inbound message contains media (image / document) | `runMediaHandlerFlow` | `$conversation`, `$contact`, `$input` (includes media descriptor), `$pre_turn`, `$inbound`, `$config`, `$vars` | No |

### Agent tool flow

The LLM-driven path. When the agent calls a tool, Mastra hands control to the platform's `FlowTool.run()`. The tool resolves its YAML, executes the entry-decision tree, then runs each step until a render-emitting step or terminal `end:` port. Steps may suspend (waiting on a user pick), in which case the run is persisted and resumed on the next tap.

Declared in `manifest.agents[].tools[]`. Agent-callable; LLM picks the tool by description.

### Pre-turn flow

Runs *before* the agent every inbound, in declaration order. Used for contact resolution, language inference, pack-domain user upsert — anything that should be settled before the LLM sees the message.

Results published via `assign:` land in `result.vars`, which the platform caches per `(tenant, channel, channel_contact_handle, flow_id)` for `cache_ttl_s` seconds and exposes to every downstream flow as `$pre_turn.<key>`. So a pre-turn flow that assigns `user_id` makes `$pre_turn.user_id` available to every tool flow + post-turn flow in the same turn (and subsequent turns within TTL).

Declared in `manifest.pre_turn_flows[]`. Cannot be called by the agent (the loader throws if any agent tool list references one).

### Post-turn flow

Runs *after* the agent emits its outbound reply. Side-effect-only — anything the post-turn flow renders is discarded (the conversation is already closed for this inbound). Used for analytics, audit logging, follow-up scheduling.

Declared in `manifest.post_turn_flows[]`. Cannot be called by the agent.

### Media handler (escape hatch)

Runs when an inbound carries a media part (image, document) and the pack opted into the custom-flow path. Most packs instead enable a built-in `inbound_preprocessing` service (vision / transcription) — no flow needed (see [`docs/19-platform-media.md`](19-platform-media.md) §6). The flow is the escape hatch for custom side-effects: download, classify, attach metadata to the conversation, sometimes hand-off to a tool flow.

Declared at `manifest.media_handlers.<media_type>`. Receives `{ media_id, media_ref, mime_type, caption, kind }` on `$input`. Mutually exclusive with `inbound_preprocessing.<kind>` (enabled). Cannot be called by the agent.

---

## Context namespaces

Flow expressions read context via `$<namespace>.<field>` syntax. The interpreter binds each namespace at flow entry; all bindings are read-only inside expressions (the only way to "write" is via `assign:` which appends to `$vars`).

| Namespace | What it carries | Source | Mutable? |
|---|---|---|---|
| `$conversation` | Universal conversation invocation context | `ConversationContext` (Mastra `RequestContext`) | No |
| `$contact` | Persisted platform Contact row | DB `contacts` table | No |
| `$input` | The flow's declared inputs | Caller-supplied per invocation | No |
| `$pre_turn` | Values published by pre-turn flows | `result.vars` of every `pre_turn` flow run this turn | No |
| `$inbound` | Per-message channel ephemera | ALS scope set by inbound dispatcher | No |
| `$config` | Pack config bundle | `manifest.config:` block | No |
| `$vars` | Flow-local working state | `assign:` accumulator | Append-only via `assign:` |
| `$param` | Template parameters | Only bound inside template bodies | No |

### `$conversation.*` — universal invocation context

What was passed for *this turn*, channel-blind:

```yaml
$conversation.channel                # 'whatsapp' | 'web' | 'slack' | 'sms' | 'voice'
$conversation.channel_contact_handle # universal handle (WA number, web session id, …)
$conversation.name                   # display name as the host knows it
$conversation.lang                   # 'ar' | 'en' | 'bilingual'
$conversation.conversation_id        # null inside pre-turn (thread not opened yet); UUID inside tool/post-turn
$conversation.tenant_id              # tenant UUID
$conversation.project_id             # project UUID
```

A flow that needs to branch on the channel declares `conversation_context.channel: { type: enum, enum: [whatsapp, web, slack, sms, voice] }` and reads `$conversation.channel`. Most flows don't — they rely on the pack-level `default_conversation_context` to declare the universal fields once.

### `$contact.*` — persistent platform contact row

What's *on file* for the speaker:

```yaml
$contact.id
$contact.channel
$contact.channel_contact_handle
$contact.display_name
$contact.language
$contact.metadata               # JSONB — namespaced
```

`metadata` follows a strict two-namespace convention:

```yaml
$contact.metadata.whatsapp.profile_name      # channel namespace — adapter-published
$contact.metadata.whatsapp.waba_phone_id     # ditto
$contact.metadata.web.user_agent             # web namespace
$contact.metadata.web.ip
$contact.metadata.real_estate.user_id        # pack namespace — pack-domain attributes
$contact.metadata.real_estate.roles
```

Bare top-level keys (`metadata.user_id`) are rejected at the write seam. See [channels reference](./16c-platform-stdlib-channels.md) for the live per-channel field list, and [src/platform/db/types.ts](../src/platform/db/types.ts) for the convention doc.

### `$input.*` — caller-supplied flow inputs

Declared via the flow's `input:` block:

```yaml
input:
  channel_contact_handle: { type: string, describe: 'Handle for this conversation.' }
  sender_name:            { type: string, optional: true }
```

Read as `$input.channel_contact_handle`. The interpreter validates required + types at flow entry; mismatched callers fail-closed before any step runs.

### `$pre_turn.*` — pre-turn published values

Whatever every pre-turn flow has published via `assign:` lands here, keyed by assign-key. The real-estate pack's `resolve-pack-user` pre-turn publishes:

```yaml
$pre_turn.user_id      # UUID — pack-domain users row id
$pre_turn.user         # full pack users row (object)
$pre_turn.display_name
$pre_turn.language
$pre_turn.roles
```

So a tool flow can read `$pre_turn.user.name_ar` without calling `get_pack_user_by_contact` again — the value was already materialised + cached upstream. Empty inside the pre-turn pipeline itself (the accumulator is still being built).

### `$inbound.<channel>.<field>` — per-message channel ephemera

Per-message data that's NOT durable on the contact row. The dispatcher's `withInbound` ALS scope makes the same blob visible to every flow type in the same turn (tool, pre-turn, post-turn, media-handler) without per-callsite threading.

```yaml
$inbound.whatsapp.message_id          # wamid
$inbound.whatsapp.referral            # CTWA referral block (when present)
$inbound.web.local_id                 # client-assigned echo id
$inbound.web.ip
$inbound.web.referer
$inbound.web.accept_language
$inbound.web.user_agent
```

Null slices mean the inbound arrived on a different channel — e.g. `$inbound.whatsapp.message_id` is null on a web inbound. See [channels reference](./16c-platform-stdlib-channels.md) for the live per-channel ephemera field list.

### `$config.*` — pack config bundle

Whatever the pack's `manifest.config:` declares. Read at flow load time and bound once per run. Used for thresholds, feature toggles, locale tables — anything the pack wants to keep out of the YAML body.

### `$vars.*` — flow-local working state

Whatever the flow's own `assign:` statements have written so far. Bare references `$vars.foo` are equivalent to `$foo` in expressions — both resolve through the same scope. `assign:` is append-only within a run; there is no `delete:` or reassignment of nested paths.

### `$param.*` — template parameters

Only bound inside the body of a template (`templates/*.template.yaml` under any platform stdlib directory). When a flow `use:`-expands a template, the template's `params:` block is satisfied from the `with:` map, and the body reads them as `$param.<key>`. Outside template bodies `$param.*` is unbound — references throw at expression-evaluation time.

---

## How `buildFlowRunContext` ties it together

Every flow-invocation site goes through one builder at [src/platform/flow-runtime/flow-run-context.ts](../src/platform/flow-runtime/flow-run-context.ts):

```ts
const { context, conversation_context } = await buildFlowRunContext({
  flow_kind:              'tool',         // or 'pre_turn' | 'post_turn' | 'media_handler'
  flow_id:                this.id,
  channel:                'whatsapp',
  channel_contact_handle: '+201234567890',
  tenant_id, project_id, conversation_id,
  sender_name, contact, pre_turn_vars,
})
await runFlow({ context, conversation_context, /* … */ })
```

That one call resolves the Contact row (or reuses a pre-supplied one), composes `$conversation` from the universal fields, reads cached `$pre_turn` vars (when applicable), and produces the exact `{ context, conversation_context }` pair `runFlow` consumes. New flow kinds drop in by adding a discriminator + calling the same builder — every namespace continues to resolve the same way across kinds.

---

## When the LLM should know about the context

`$conversation.*` and `$contact.metadata.*` are flow-expression-readable but the **LLM does not see them directly**. The lever to surface a fact to the LLM is **working memory** — the Mastra-native durable per-thread slot Mastra auto-prepends to the system prompt every turn.

The platform's default working-memory populator at [src/platform/agent/runs/run-agent-turn.ts](../src/platform/agent/runs/run-agent-turn.ts) writes `channel` + `language` + contact name into the slot. Packs that need per-conversation custom facts override `manifest.agents[].working_memory.populate(session, channel)` to add their own. Memory notes (the `note:` flow-runtime node — lineage: retired `inject_memory_note` primitive → `emit_envelope.memory_note` → `note:`) are the right lever for *episodic* nudges ("the user just declined an offer — be tactful"); working memory is the right lever for *stable* per-conversation context.

See [schemas/context.md](../src/platform/flow-runtime/schemas/context.md) §11b for the `note:` pattern and its per-flow-kind delivery (tool flows → `tool_action`; pre-turn/media flows → `PreTurnBriefingProcessor`), and [20-two-conversation-model.md](./20-two-conversation-model.md) §3.4 for the envelope contract.
