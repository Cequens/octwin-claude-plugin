# Platform Runtime — Taps Reference

> 📝 **Hand-curated.** Edit this file directly when changing the tap schema, adding a match kind, or introducing a new dispatch verb. The Reference page in the operator console renders it verbatim — changes are picked up at the next request (60s cache).

A **tap** is any inbound user action that isn't free text — quick-reply buttons, list-row selections, carousel card buttons, CTWA referrals, and structured deep-links. Taps land at the platform's dispatcher and resolve to a typed `ResolvedTap` that names the target tool, the input shape, and the LLM-visible agent token.

There are **two ways** a tap resolves:
1. **Structured ids** — render-owned `t:<verb>:<target>:<bindings>` ids the renderer emitted from a flow's `on_select`. No declaration needed.
2. **Declared taps** — pack-level `taps.yaml` matches an external id format (`portal_offer:<uuid>`, CTWA buttons, etc.).

Structured ids win when both could match — they're explicit and carry their dispatch in the id.

Companion docs:
- [DSL reference](./13a-dsl-reference.md) — `on_select` syntax inside flow YAML
- [Flow types & context](./16d-flow-types-and-context.md) — how dispatched taps reach a flow
- [Channels](./16c-platform-stdlib-channels.md) — channel-specific tap surfaces

---

## `taps.yaml` — pack-level declarations

Lives at `src/packs/<pack-id>/taps.yaml`. Loaded once at boot; the loader compiles each declaration into a `TapDeclareSpec` and registers it with the platform's `TapRegistry`. Pack-level taps are preferred over per-flow `taps:` blocks (the schema permits both but no real-estate flow uses the latter today).

### Shape

```yaml
declarations:
  - surface:     button | list | either | text | ctwa   # default: button
    tool_id:     <string>                                # optional; defaults to pack id
    match:       <match-spec>                            # required
    dispatch:    agent | direct | workflow | text       # required
    tool:        <flow-id>                               # required when dispatch=direct
    input:       { <key>: <template> }                  # optional; passed to the tool as input
    agent_token: <template>                              # required for dispatch != text
    text_reply:  <string>                                # required for dispatch=text
    describe:    <string>                                # human-readable; surfaces in logs
    priority:    <number>                                # default 0; higher matches first
```

The schema is strict — unknown top-level keys are rejected at boot with a Zod parse error naming the offending field.

### Match kinds

| Kind | Schema | Matches against | Semantics |
|---|---|---|---|
| `id` | `{ id: 'view_my_leads', against?: <field> }` | Button / list-row id, OR the named field on `ctwa` surfaces | Exact equality. Use for static UI ids. |
| `prefix` | `{ prefix: 'portal_offer:', against?: <field> }` | Same as `id` | Starts-with. The remainder after the prefix becomes `$arg` in templates. |
| `regex` | `{ regex: '^ctwa_compound(?::(.+))?$', against?: <field> }` | Same | Case-insensitive regex. Capture groups expose as `$1..$N`. |
| `text_regex` | `{ text_regex: 'رقم العقار[:：]\s*(.+)' }` | Inbound free-text body | Promotes a text inbound to a tap. The full id stays the text body; `$arg` is empty. |
| `default` | `{ default: true }` | Any surface | Catch-all. Lowest priority. |

`against:` lets a `ctwa` declaration match a non-default referral field — e.g. `against: source_url` (the default), `against: headline`, `against: source_id`. Without `against:`, `ctwa` matches against `source_url`.

### Surfaces

| Surface | What it matches | Common dispatch |
|---|---|---|
| `button` | Quick-reply buttons (WhatsApp interactive `button_reply`) | `direct`, `workflow`, `agent` |
| `list` | List-row selections (WhatsApp interactive `list_reply`) | `direct`, `workflow`, `agent` |
| `either` | Both `button` and `list` — single declaration covers both wire shapes | Same as above |
| `text` | Free-text inbound body that matches `text_regex` | `agent` only today |
| `ctwa` | CTWA referral payload field | `agent` only today |

Today `text` and `ctwa` only support `dispatch: agent`. The loader throws at boot if a pack tries `dispatch: direct` on those surfaces — a follow-up wave will add it.

### Dispatch verbs

| Dispatch | What happens | When to use |
|---|---|---|
| `agent` | Platform synthesises a user message from `agent_token`, posts it to the Mastra thread, lets the LLM pick the tool. | The default. Use when the tap should look "as if the user typed it" so subsequent free-text turns keep coherent context. |
| `direct` | Platform calls the named `tool:` flow's `run()` directly, no LLM. Mirror writes a typed user message into the thread so the next turn sees the history. | When you know exactly which flow handles the tap and want to skip the LLM round-trip (latency / token cost). |
| `workflow` | Resume the tool's suspended workflow (was a `suspend:` step waiting on user input). No LLM. | When the inbound is the resume signal for an open multi-step flow (offer-confirm, listing-edit, etc.). |
| `text` | Send `text_reply` as a static outbound message. No tool, no LLM. | Hard-coded short replies (FAQ buttons, "we'll get back to you"). |

**Direct-tap rules** (mind these — three live feedback memories cover the failure modes):
- Direct taps land in thread memory automatically: `runDirectTap` wraps `agent.generate()` (via synthetic dispatch), which records the user message itself, so the next free-text turn sees the tap. The old manual `injectUserMessage(...)` mirror was retired in Phase 13.3a. ([feedback_mirror_direct_taps_to_thread](../memory/feedback_mirror_direct_taps_to_thread.md))
- When a direct tap fires a `suspend:` step (gap render), persist `workflow_run_id` + `instructions` to thread memory. Otherwise the next typed turn can't resume. ([feedback_direct_tap_gap_must_persist_runid](../memory/feedback_direct_tap_gap_must_persist_runid.md))
- A direct-tap whose tool returns no UI (`_render` / `message_ar` both empty) falls through to the LLM loop. The mirror is skipped on that path because `generate()` records the user message itself. ([feedback_direct_tap_escalates_to_llm](../memory/feedback_direct_tap_escalates_to_llm.md))

### Template vocabulary

Used inside `agent_token:` and each value of `input:`. Substitution happens at match time, not at boot.

| Token | Meaning |
|---|---|
| `$arg` | Raw id minus the matched prefix. Empty when `match.kind=regex` (use `$1..$N` instead). |
| `$id` | Full inbound id verbatim. |
| `$title` | Visible button / list-row title (empty for `text` / `ctwa` surfaces). |
| `$1` … `$N` | Regex capture groups (only when `match.kind=regex`). |
| `$opt( … )` | Renders the inner text **only when every `$N` / `$<field>` reference inside is non-empty**. Use to fold optional captures into a sentence without an `if`. |
| `$slice( $ref, N )` | First N characters of the resolved reference. Useful inside `$opt(...)` to cap a free-form field (e.g. `$slice($headline, 50)`). |
| `$source_url` / `$source_id` / `$headline` / `$body` / `$image_url` / `$media_type` | CTWA referral fields. Available on `ctwa` surfaces only; empty string elsewhere. |
| `$$` | Literal `$`. |

Unknown placeholders throw at boot — the loader names every `$<token>` it can't resolve in the offending template.

### Examples (real-estate pack)

```yaml
# Portal deep-link: id is `portal_offer:UID-12345678`.
# $arg becomes UID-12345678 → LLM sees the typed marker + Arabic prompt.
- match:       { prefix: 'portal_offer:' }
  dispatch:    agent
  agent_token: '[PROPERTY_ACTION:$arg:offer] أريد تقديم عرض سعر على الوحدة'
  describe:    'Portal deep-link: offer'

# CTWA campaign button: id like `ctwa_compound` or `ctwa_compound:NewCairo`.
# $opt(:$1) only renders ":NewCairo" when the suffix is present.
- match:       { regex: '^ctwa_compound(?::(.+))?$' }
  dispatch:    agent
  agent_token: '[CTWA_ACTION:compound$opt(:$1)] أريد معلومات عن الكمباوند$opt( ($1))'
  describe:    'CTWA campaign button: ctwa_compound'

# Free-text portal reference: user types "رقم العقار: UID-12345678".
# text_regex promotes the inbound to a tap, primes the LLM with [PROPERTY_REF:].
- surface: text
  match:   { text_regex: 'رقم\s*العقار\s*[:：]\s*([0-9a-f-]{36}|UID-[0-9A-Fa-f]{8})' }
  dispatch: agent
  agent_token: '[PROPERTY_REF:$1]'
  describe: 'Arabic portal deep-link in free text'

# CTWA referral with structured id in the source URL.
- surface: ctwa
  match:   { regex: '[?&](?:property|id)=([0-9a-f-]{36})', against: source_url }
  dispatch: agent
  agent_token: '[CTWA_CONTEXT:type=property|id=$1]'
  describe: 'CTWA: property uuid from ad URL'
```

---

## Structured tap ids — `t:<verb>:<target>:<bindings>`

The render-owned wire format. When a flow's `on_select` declares a dispatch, the renderer encodes it into the button / list-row id at emit time; no `taps.yaml` declaration needed at the receiving end. The decoder synthesises a `TapDeclaration` on the fly so the rest of the pipeline is uniform.

### Verbs

| Verb | What it does | Example id |
|---|---|---|
| `t:invoke:<flow-id>:<bindings>` | Call the named flow's `run()` directly with the decoded bindings as input. | `t:invoke:property-search:compound_id=abc123` |
| `t:resume:<flow-id>:<bindings>` | Resume the flow's suspended workflow (matched by `workflow_run_id` in the bindings). | `t:resume:offer-confirm:run_id=...&approved=true` |
| `t:reply:<text>` | Send `<text>` as a static outbound message. Equivalent to `dispatch: text` declared taps. | `t:reply:Thanks!` |
| `t:agent:<message>` | Inject `<message>` as a user message into the Mastra thread. The LLM picks the next tool. | `t:agent:show me listings under 5M` |

Bindings are URL-encoded `key=value` pairs separated by `&`. The wire format is bounded by WhatsApp's **256-byte button id cap** — the encoder throws `TapIdError` at flow-load time if a binding would overflow. Keeping bindings short (use `UID-XXXXXXXX` refs, not full UUIDs when possible) is the practical workaround.

### When to use structured vs declared

| Question | Answer |
|---|---|
| Is the dispatch decided by the flow's own `on_select`? | Structured id. The renderer owns the encoding. |
| Is the id format external (CTWA campaign URL, portal app, third party)? | Declared via `taps.yaml`. The pack has to recognise an id it didn't generate. |
| Is the same flow accessible from multiple wire-shaped ids? | Declared, with `priority:` to control match order. |
| Is the dispatch dynamic per-row (e.g. each carousel card targets a different flow)? | Structured. Build the id in YAML — `on_select: 'invoke:lead-action?action=offer&ref=$row.uid'`. |

---

## Resolution lifecycle

1. **Inbound arrives** — interactive button/list payload, CTWA referral, or free text.
2. **Structured-id check** — `isStructuredTapId(rawId)` matches `t:<verb>:…`. If yes, decode + synthesise a `ResolvedTap`.
3. **Declared-tap loop** — sort declared taps by `(priority desc, declaration order asc)`. First `tryMatch` to return non-null wins.
4. **Dispatch** — the handler routes by `decl.dispatch`:
   - `agent` → write user message to thread, agent picks tool
   - `direct` → call `tool.run(input)`, mirror result, follow direct-tap rules above
   - `workflow` → resume `tool.workflow` keyed by `workflow_run_id`
   - `text` → send `textReply` outbound, log, done
5. **Audit** — every resolved tap writes a `conversation_events.tap` entry naming surface / match-kind / dispatch / tool / agent token (truncated). Visible in the operator timeline.

---

## Picker-id conventions (UI authoring note)

> Tap routing is **declarative**, not prefix-based. Render-owned taps carry a structured id (`t:<verb>:<target>:<bindings>`) that names the verb + target tool explicitly, and tools register `TapDeclaration` patterns with `TapRegistry`; `TapRegistry.resolve(id, …)` does the routing. The old stateless `wa-response-resolver` prefix-match — where unprefixed ids defaulted to `property-search` — is **retired**. ([feedback_picker_id_prefix_rule](../memory/feedback_picker_id_prefix_rule.md))

The `cmp:`/`lead:`/`pub:` strings survive only as an author-side **id convention** (the real-estate pack uses them so an operator inspecting a WhatsApp payload can read what a row does; the matcher table above can still match on `prefix`). They no longer decide routing — register a `TapDeclaration` on the tool for that. An unmatched id warn-logs and falls through to a legacy `[PROPERTY_SELECTED:id]` token, which rarely fires because `property-search` owns the `default` declaration.

---

## Tool-side: skipping free-text disambig

Tools that serve **both** the LLM (free-text intent) and direct-tap callers must gate their fuzzy-disambig branches on an `inputs_resolved` flag the tap declaration sets. Without it, picker selections loop back into disambig forever. ([feedback_tap_inputs_skip_freetext_disambig](../memory/feedback_tap_inputs_skip_freetext_disambig.md))

Typical shape in the tool's flow YAML:

```yaml
entry:
  - if: '$input.inputs_resolved == true'
    goto: resolved_path
  - else: true
    goto: disambig_path
```

The tap declaration's `input:` map sets it:

```yaml
- match:    { prefix: 'lead:select:' }
  dispatch: direct
  tool:     lead-action
  input:
    lead_id:         '$arg'
    inputs_resolved: 'true'
  agent_token: '[LEAD_SELECTED:$arg]'
```

---

## Source files

- [src/platform/taps/types.ts](../src/platform/taps/types.ts) — `TapDeclaration`, `TapMatch`, `ResolvedTap`, `TapDispatch`
- [src/platform/taps/registry.ts](../src/platform/taps/registry.ts) — runtime registry + resolve loops
- [src/platform/taps/matchers.ts](../src/platform/taps/matchers.ts) — priority/ordinal-aware match logic
- [src/platform/taps/structured-resolver.ts](../src/platform/taps/structured-resolver.ts) — `t:<verb>:…` decoder
- [src/platform/taps/id-codec.ts](../src/platform/taps/id-codec.ts) — `encodeTapId` / `decodeTapId` + 256-byte cap enforcement
- [src/platform/manifest/taps.ts](../src/platform/manifest/taps.ts) — YAML schema + template compiler
- [src/packs/real-estate/taps.yaml](../src/packs/real-estate/taps.yaml) — live example
