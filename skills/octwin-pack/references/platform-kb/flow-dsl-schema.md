# Flow YAML Schema Reference

> **For authoring agents.** This document is the canonical reference for
> the flow-runtime YAML schema. Embed it in any agent prompt that
> generates, validates, or modifies `*.flow.yaml` files. Auto-generated
> snapshot of the JSON Schema is appended at the bottom; the prose above
> explains semantics the JSON Schema can't capture.
>
> **Do not paraphrase examples.** Copy verbatim from the example flow at
> the end. Schema-violating outputs fail at load time.

## Top-level structure

A flow is one YAML document with these top-level keys. Required ones marked ✱.

| Key | Type | Notes |
|---|---|---|
| `flow_id`✱ | string | Becomes the public tool id. Must match the file name (e.g. `home.flow.yaml` → `flow_id: home`). Lowercase letters, digits, dashes; starts with a letter. |
| `version`✱ | int | Bumped on breaking changes. Used by `resumeFlow` to reject stale `run_state`. |
| `description` | string \| multiline | Becomes the Mastra tool description seen by the LLM. |
| `constants` | record | Static values used by expressions (e.g. enum sets). Available as `$<NAME>`. |
| `config` | object | Per-flow runtime knobs grouped by category (`thresholds`, `retries`, `models`, …) — see §7. |
| `input` | record | Field schema → input fields the agent passes — see §2. |
| `conversation_context` | record | Ambient channel / agent context the flow expects (channel_contact_handle / lang / tenant_id …). Same FieldSpec shape as `input:`; resolved from Mastra `RequestContext` at entry; exposed as `$conversation.X` in expressions — see §2b. |
| `taps` | array | Inbound tap declarations (button/list ids → tool input) — see §6. |
| `entry`✱ | array | Routing rules from arrival to first step — see §3. |
| `steps`✱ | record | Named step bodies (each is a node or array of nodes) — see §4. |

```yaml
flow_id: home
version: 1
description: Welcome / status entry for the current user.

input:
  user_id: { type: string, optional: true }

entry:
  - else: true
    goto: render

steps:
  render:
    # Pack User row is published by the pre-turn pipeline (see
    # `manifest.pre_turn_flows:`). Read from `$pre_turn.user`
    # directly — no per-tool lookup roundtrip.
    - do:   get_home_summary
      args: { user: '$pre_turn.user' }
      bind: summary
    - render:
        type: buttons
        body: 'مرحباً {$summary.first_name} 👋'
        buttons: [{ id: ok, title: تمام }]
    - end: applied
```

## §2 Input fields

Map of `<field_name>: <FieldSpec>`. The loader compiles these into a
real Zod schema for Mastra tool registration.

```yaml
input:
  property_id:
    type: string
    optional: true
    describe: 'Property UUID or unit reference (UID-XXXXXXXX).'
  bedrooms:
    type: number
    integer: true
    min: 0
    optional: true
  type:
    type: enum
    enum: [apartment, villa, townhouse, twinhouse, duplex, penthouse, studio, chalet]
    optional: true
  amenities:
    type: array
    items: { type: string }
    optional: true
  photo_analyses:
    type: array
    items:
      type: object
      shape:
        url:      { type: string }
        analysis: { type: object, optional: true }
    optional: true
```

Fields:

| Field | Applies to | Notes |
|---|---|---|
| `type`✱ | all | One of `string \| number \| boolean \| enum \| array \| object`. |
| `optional` | all | Default false. Required by default — flag explicitly to relax. |
| `nullable` | all | Allows `null` as a valid value. |
| `positive`, `integer`, `min`, `max` | number | Zod constraints. |
| `enum` | enum | Required for `type: enum`. |
| `items` | array | Sub-FieldSpec for elements. |
| `shape` | object | Map of `<key>: <FieldSpec>` for nested objects. |
| `describe` | all | Becomes Zod's `.describe()` — surfaced to the agent. |

## §2b Agent context

Ambient channel / agent context the flow expects from its host. Same
`FieldSpec` shape as `input:`. The runtime resolves these from
Mastra's `RequestContext` at every flow entry; missing required
fields cause the flow to fail-closed before any step runs.

```yaml
conversation_context:
  channel_contact_handle: { type: string, describe: "Sender's channel handle (WhatsApp E.164 / web `from` / …)." }
  lang:      { type: enum, enum: [ar, en, bilingual], default: ar }
  tenant_id: { type: string, optional: true }
```

In expressions:
- `$conversation.channel_contact_handle` — channel handle of the sender
- `$conversation.lang`     — UI language

Why conversation_context vs input:
- **`input:`** = arguments the agent passes when invoking the tool
  (per-call). Validated against the agent's tool-call shape.
- **`conversation_context:`** = ambient context the flow infers from its
  host. Validated against `RequestContext`. Same field-spec, different
  source.

Pack-level defaults (in `pack.ts → manifest.default_conversation_context`)
are merged BEFORE per-flow `conversation_context:`; flows extend or
override pack defaults per-key.

## §3 Entry routing

An ordered array. The interpreter picks the first matching entry to
land on a step.

```yaml
entry:
  - if:   '$input.action != null'
    goto: validate
  - if:   '$input.lead_id != null'
    goto: render_detail
  - else: true
    goto: render_list
```

Rules:
- Each item has either `if: <expression>` + `goto:` or `else: true` + `goto:`.
- The fall-through `else` MUST exist (otherwise unmatched entries throw at runtime).

## §4 Steps & nodes

`steps` maps step names → step bodies. A body is either a single node
or an array of nodes executed in order. Step bodies MUST end with a
terminal node (`goto`, `end`, `answer`, or `pause`); falling through is
a runtime error.

### Node operations (exactly one per node)

| Key | Purpose |
|---|---|
| `do` | Call a registered primitive. Common modifiers: `args`, `bind`, `outputs`, `detach`. |
| `if` | Conditional. Pairs with `then` / `else`, OR the `{ if, goto }` direct-jump sugar (goto XOR then; `else` still allowed). |
| `branch` | Multi-case. Each case has `when:` or `default: true` + `then`. |
| `assign` | Set vars. Body: `{ <var>: <expression> }`. |
| `render` | Resolve a UiHint and attach to envelope. |
| `pause` | Park the run waiting for the next user input (typed text or interactive tap). Body: `{ on_resume }` (a breadcrumb rides the preceding render's `memory_note:`). The resume payload is this turn's `$input`, folded into `$state` by the turn-merge. |
| `goto` | Transition to another step. |
| `end` | Terminate flow. Body: `'applied' \| 'aborted' \| 'failed'` (or `{ status, body }` to render a text_card first; the `publish:` sibling publishes vars/cache — renamed from `with:`). |
| `answer` | **Terminal** "agent composes the reply from tool data". Body: `{ data?, instructions }` — merges `data`, sets a short `instructions` brief, emits NO render (agent composes this turn), ends `applied`. `.strict()`: no `memory_note`/render. |
| `note` | Non-terminal breadcrumb: push ONE memory-note string onto the run's `memory_notes` list, no card. String value, `$expr`/`{$expr}` both resolve. (Formerly `emit_envelope`, whose `data`/`instructions` fields were retired — use `answer:`/`render:` for those.) |

### Modifiers / siblings

Allowed alongside the operation key:

| Sibling | Valid on | Purpose |
|---|---|---|
| `args` | `do` | Args object passed to the primitive. Values are expressions. |
| `bind` | `do` | Bind result to `$<bind>` for later expressions. |
| `outputs` | `do` | **Output-port switch table** for primitives that declare typed `outputs: { port: schema }` via `definePrimitive`. Map of port name → sub-sequence — the interpreter routes to whichever port the handler returned. See §4b. (A null result is handled with `bind` + `if: '$<bind> == null'`, or a `not_found` port — the old `on_null` sibling was retired.) |
| `detach` | `do` | Fire-and-forget: kick the primitive off without awaiting (best-effort side effects). No result → mutually exclusive with `bind`/`outputs`. |
| `then` | `if` | Sub-sequence on truthy guard. |
| `goto` | `if` | Direct-jump sugar — `{ if, goto }` ≡ `{ if, then: [{ goto }] }`. XOR with `then`. |
| `else` | `if` | Sub-sequence on falsy guard. |
| `on_resume` | `pause` | Sub-sequence on resume; reads this turn's `$input` (the resume/tap payload, already folded into `$state`) + `$control`. (The retired `$resume` channel was the pre-fold staging of `$input`.) |
| `memory_note` | `render` | Breadcrumb attached to envelope (interpolation supported). |
| `data` | `render` | Extra fields merged into envelope `data`. Object literal or `'$expr'` returning object. |
| `instructions` | `render` | Agent-instructions override (e.g. render-less `lookup_only` ports). |

### §4b Output ports — the canonical multi-outcome shape

Primitives declare typed output ports via `definePrimitive({ outputs: { port: zodSchema } })`. The flow consumes them with an `outputs:` switch table:

```yaml
- do:   validate_lead_action
  args: { buyer_channel_contact_handle: '$conversation.channel_contact_handle', action: '$input.action', lead_id: '$input.lead_id' }
  bind: validation
  outputs:
    fail:  [{ render: { type: text, body: '$validation.reason' } }, { end: failed }]
    gap:   [{ render: { type: text, body: '$validation.prompt' } }, { pause: { on_resume: [{ goto: validate }] } }]
    ready: [{ goto: apply }]
```

Rules:
- Each declared port MUST appear (or be exhaustively covered by a `default` port — TBD).
- Inside a port's branch, `$<bind>` is the union, but the runtime narrows to that port's typed payload (TS-side via the schema; YAML treats it as an object).
- Express "not found" as a port (e.g. `not_found:`) rather than a null-check.

Output ports replaced an older `if: '$result.path == "..."'` branching pattern. New flows should always use ports.

### Examples

```yaml
# do + bind, then null-check
- do:   get_user_by_contact_handle
  args: { channel_contact_handle: '$input.seller_channel_contact_handle' }
  bind: seller
- if: '$seller == null'
  then:
    - render: { type: buttons, body: 'لم يتم العثور.', buttons: [{ id: ok, title: تمام }] }
    - end: failed

# if / then / else
- if: '$input.action != null'
  then: [{ goto: validate }]
  else: [{ goto: render_list }]

# if / goto — conditional direct jump (sugar for then: [{ goto }])
- if: '$seller == null'
  goto: render_no_match

# branch
- branch:
    - when: '$kind == "manage"'
      then: [{ goto: apply_manage }]
    - when: '$kind == "update"'
      then: [{ goto: apply_update }]
    - default: true
      then: [{ end: failed }]

# assign (multiple vars)
- assign:
    shown_count: '$views.length > 10 ? 10 : $views.length'
    truncated:   '$views.length > 10'

# render with all siblings
- render:
    type: list
    header: '🛠️ {$view.reference}'
    body:   '$card_body'
    sections: [{ rows: '$rows' }]
  memory_note: 'Showed manage card for {$view.reference}'
  data:        { status: manage, reference: '$view.reference' }
```

### Pause / on_resume

`pause:` is a terminator marker (like `end:`) that parks the run waiting for
the next user input. Renders + envelope-shaping live on PRECEDING `render:`
nodes — pause itself holds only resume mechanics. The resume payload IS this
turn's `$input` (and is folded into `$state` by the turn-merge before any node
runs), so `on_resume` reads `$input.<field>` / `$state.<field>` / `$control.*`.
Read `$state` for values that must survive across turns; `$input` is per-turn.

Approve / decline pattern (typed buttons emit structured `resume:$self` taps):

```yaml
- render:
    render_intent: detail_card
    body:    '$preview_body'
    buttons:
      - { title: 'تأكيد ✅', on_select: { resume: $self, with: { approved: true } } }
      - { title: 'إلغاء ❌', on_select: { resume: $self, with: { approved: false } } }
- pause:
    on_resume:
      - if: '!$input.approved'
        then: [{ goto: aborted }]
      - goto: apply
```

Gap-style (typed reply from user — agent or user supplies a missing field):

```yaml
- render:
    render_intent: text_card
    body: '$validation.prompt'
  instructions: '$validation.instructions'
- pause:
    on_resume:
      - goto: validate     # validate re-reads $state.offer_amount_egp (folded from this turn's $input)
```

## §5 Render specs

Three forms accepted:

**A. Inline UiHint** — must include `type:`. All other fields can be
expressions.

```yaml
render:
  type: list
  header: '🏘️ {$compound}'
  body:   '$body_text'
  footer: 'اختار من القائمة.'
  button_text: 'اختار'
  sections:
    - rows:
        for_each: '$take($items, 10)'
        as: item
        template:
          id:    'cmp:{$item.name_en}'
          title: '{$item.name_ar ?? $item.name_en}'
          description: '{$item.developer ?? ""}'
```

**B. Variable reference** — primitive returns the full UiHint.

```yaml
- do: build_compound_list_render
  args: { items: '$items' }
  bind: render_spec
- render: '$render_spec'
```

**C. Template ref (escape hatch)** — calls a TS function from the
templates registry. Use sparingly — most renders should be inline.

```yaml
render:
  _from: build_diff_body
  args: ['$reference', '$before', '$changes']
```

### Render fragments inside inline specs

Two recursive constructs work at any depth in an inline render:

#### Conditional fragment: `{ if, then, else }`

```yaml
sections:
  - rows:
      - { id: edit_listing, title: 'تعديل ✏️' }
      - if: '$is_live'
        then:
          id:    'unlist_listing:{$ref}'
          title: 'سحب الوحدة 🗑️'
        else:
          id:    'relist_listing:{$ref}'
          title: 'إعادة نشر 📤'
```

If `else` is omitted and the guard is false, the fragment evaluates to
`undefined` and is dropped from its parent array (or object key).

#### Iteration fragment: `{ for_each, as, template }`

```yaml
sections:
  - rows:
      for_each: '$take($views, 10)'
      as: l
      template:
        id:          'listing:{$l.reference}'
        title:       '{$l.short_title}'
        description: '{$l.type} | {$l.price_egp_formatted}'
```

The expression after `for_each:` must resolve to an array. Each item
binds to `$<as>` (defaults to `$item`); `$<as>_index` holds the 0-based
index.

## §6 Tap declarations

> **Per-flow `taps:` was retired.** Flow YAML no longer carries a
> `taps:` block. Two paths replace it:
>
> 1. **Structured tap-ids** (`t:<verb>:<target>:<bindings>`) — render
>    intents encode `on_select: { invoke | resume | reply | agent, with }`
>    at compile time. Most taps work this way with zero declaration.
> 2. **Pack-level `<pack>/taps.yaml`** — for external id schemes the
>    platform can't pre-encode (portal-app deep-links, CTWA referrals,
>    free-text regex patterns). See [16e-taps-reference.md](../../../../docs/16e-taps-reference.md).

## §7 Thresholds

Per-flow runtime knobs consumed by primitives — declared under the
generic `config:` namespace, grouped by category. Fully dynamic: any
top-level key is a category, any value inside is a free-form key→value
bag. No platform-side schemas; primitives guard reads with the
`?? <default>` idiom so missing or misspelled keys fall back to safe
defaults.

```yaml
config:
  thresholds:
    list_page_size:         10    # WA list UI cap
    cascade_short_circuit:  10    # ≤N → skip picker, dump to carousel
    result_limit:           20    # similarity-search top-N
    similarity_min:         0.75  # min cosine distance for primary results
    alt_similarity:         0.60  # min for the "alternatives" widening pass
    alt_limit:              5
    title_max_len:          24    # WA list-row title truncation cap
    idempotency_window_min: 30
  # pack-defined categories — no registration, no schema
  retries: { default_max: 3, backoff_ms: 500 }
  models:  { summarize: claude-haiku-4 }
```

Resolved into the runtime as `$config.<category>.<key>` (expressions)
and `ctx.config.<category>.<key>` (primitives). To add a new category,
just write the YAML and read it from the primitive — no platform
changes needed.

> **Removed (D5):** `tool_behavior:` (per-path control modes / button
> labels / error_messages) is gone. Behaviour is now expressed via
> per-node fields on the `render` operation itself (`memory_note:`,
> `data:`, `instructions:`) — see §4's modifier table. Suppression mode
> is the implicit default for renders that emit a UiHint; complete mode
> applies when the agent's prose is the body.

## §8 Expression DSL

Strings starting with `$` (or containing `{$expr}` interpolation) are
evaluated by the runtime against `vars`.

### Built-in vars

| Var | Source |
|---|---|
| `$input` | THIS turn's inbound delta — tool call args, or the resume/tap `with` payload. Ephemeral (per-turn); read `$state` for cross-turn values |
| `$state` | The run's persistent accumulator (`state = { ...state, ...input }` each turn). A `collect` node accumulates its fields directly into `$state` |
| `$contact` | Platform-resolved contact row (`{ id, channel, channel_contact_handle, display_name, language, metadata, ... }`). For pack-domain User fields, read from `$pre_turn.*` — published once per inbound by the pack's `pre_turn_flows` pipeline. `metadata` is NAMESPACED — channel-published attributes live under `$contact.metadata.<channel>.<field>` (e.g. `$contact.metadata.whatsapp.profile_name`), pack-domain attributes under `$contact.metadata.<pack_id>.<field>` (e.g. `$contact.metadata.real_estate.user_id`). Per-channel field rosters are auto-generated at [`docs/16c-platform-stdlib-channels.md`](../../../../docs/16c-platform-stdlib-channels.md). |
| `$pre_turn` | Merged `result.vars` blob from every pack `pre_turn_flow`. Pack-defined shape; typically includes `user_id`, `user`, `language`, `roles`. |
| ~~`$resume`~~ | RETIRED — the resume payload is now this turn's `$input` (folded into `$state`); `on_resume` reads `$input`/`$state`/`$control` |
| `$run_id` | Current workflow run id (auto-injected into `on_select: resume:` tap-ids) |
| `$tool_id` | Public tool id (current flow's `flow_id`) |
| Any `bind:` from earlier nodes | `$<bind_name>` |
| Any `assign` keys | `$<assigned_var>` |
| Any `constants:` keys | `$<CONSTANT_NAME>` (uppercased by convention) |
| `$ctx` | Tap context (only in tap declarations) |

### Operators

| Class | Tokens |
|---|---|
| Arithmetic | `+ - * / %` |
| Comparison | `== != === !== < <= > >=` (loose `==`/`!=` — `null == undefined` is true) |
| Logical | `&& \|\| ??` (short-circuit; `??` only triggers on null/undefined) |
| Unary | `! -` |
| Conditional | `cond ? then : else` |
| Member | `a.b`, `a[0]`, `a["key"]` |
| Call | `$f(arg1, arg2)` |

### Built-in functions

| Function | Example |
|---|---|
| `$is_empty(v)` | True for null/undefined/`''`/`[]`/`{}`. |
| `$has_any_key(obj, keys)` | True if obj has any of the keys defined. |
| `$has_all_keys(obj, keys)` | True if obj has all keys defined. |
| `$keys(obj)` | `Object.keys()` defensively. |
| `$length(v)` | Length for strings/arrays, key count for objects. |
| `$merge(a, b)` | Shallow object merge (b overrides a). |
| `$pick(obj, keys)` | Subset of obj. |
| `$slice(arr, start, end?)` | `Array.prototype.slice`. |
| `$take(arr, n)` | First N elements. |
| `$enum_lookup(value, map, fallback?)` | Translation-table lookup. |
| `$format_number(n, locale?)` | `Intl.NumberFormat`; default `en-EG`. |
| `$unit_ref(uuid)` | UUID → `UID-XXXXXXXX`. |
| `$lead_ref(uuid)` | UUID → `LID-XXXXXXXX`. |
| `$loc(v)` | A localized xrm field bag (`{ en?, ar? }`) → the viewer-locale string (locale → en → ar → first); scalars pass through. |
| `$record_url(record_id)` | The ABSOLUTE public page URL for an xrm record (`/r/:id`; the entity must declare `public:` in xrm.yaml). |

### Interpolation

Strings with embedded `{$expr}` are interpolated:

```yaml
body: 'مرحباً {$summary.first_name}، عندك {$summary.listings_count} وحدة.'
```

The parser matches `{...}` (NOT `${...}`). Interpolation supports
ternaries: `'{$truncated ? "معروض أحدث " + $shown_count : "كل الوحدات"}'`.

### String literals

YAML strings need careful quoting for expressions:

- Single-quote the whole expression: `'$expr'` or `"'string literal'"`.
- A literal string passed as an arg/expression: wrap in extra quotes:
  `"'cancel'"` or `'"cancel"'`.

## §9 Common patterns

### Resume-then-loop (multi-pause collect)

The resume payload is this turn's `$input`, and the turn-merge folds it into
`$state` before the target step re-runs — so the accumulator is always current
with NO manual `$merge`. Authors design picker `on_select.with` payloads as flat
field-updates so they accumulate into `$state`; read `$state.<field>` for values
that must survive the suspend. (The `collect:` node is the first-class form of
this loop — the manual shape below is for flows that route their own picker.)

```yaml
manual_loop:
  - do:   analyze_state
    args: { state: '$state' }        # $state already holds this turn's delta (turn-merge)
    bind: analysis
    outputs:
      ready: [{ goto: approve }]
      field_picker:
        - render: '$analysis.render'        # list_picker with for_each
        - pause:
            on_resume: [{ goto: manual_loop }]
```

### Custom picker row (workflow-routed)

Use `on_select: { resume: '$tool_id', with: { <field>: <value> } }` on a
list row / button. The renderer encodes a structured tap-id (`t:resume:
<tool>:run_id=<run>;<field>=<value>`); the response resolver decodes it
into a flat resume payload `{ <field>: <value> }` and routes to
`handleWorkflowResume` — no LLM call.

```yaml
- title: '{$compound.name_ar}'
  on_select:
    resume: '$tool_id'
    with:   { kind: 'field', field: 'compound', value: '$compound.name_en' }
```

`$run_id` is auto-injected by the renderer; authors don't pass it.

### Render-less completion (lookup_only)

```yaml
- do: property_search_run
  args: { input: '$input', contact: '$contact' }
  bind: result
- render:       '$result.render'        # may resolve to null
  data:         '$result.data'
  instructions: '$result.instructions'
  path:         '$result.path'
- end: applied
```

When `$result.render` is null, no `_render` is set; only `data` +
`instructions` flow to the envelope.

## §10 Validation contract

Every flow YAML is loaded by `loadFlow()` and runs:

1. Zod parse against the schema (this document's structure).
2. `superRefine` — node operation exclusivity, required siblings.
3. Expression syntax validation — every expression-position string is
   parsed by jsep; failures throw with file + path + reason.

Loader self-test: `npx tsx src/flow-runtime/loader.ts` validates every
`src/flows/*.flow.yaml`. CI runs this; any new flow must pass cleanly.

## §11 Localization

User-facing copy lives in sibling locale files, not inline in the flow
YAML. This separation is required for multi-locale tenants and is the
contract the Render/UX agent uses to author copy in any language.

### File layout

For each flow, create one locale file per supported language:

```
src/flows/
  home.flow.yaml
  home.locale.ar.yaml
  home.locale.en.yaml
  …
```

### Locale file shape

```yaml
flow_id: home

strings:
  card_header:        '🏡 عمار — مساعدك العقاري'
  greeting_with_name: 'أهلاً يا {first_name} 👋'
  greeting_default:   'أهلاً بك 👋'
  status_listings:    '📋 وحداتك المسجلة: {count}'
```

- Keys are flat under `strings:`. No nesting.
- Placeholders use `{name}` syntax — substituted from the second arg of
  `$t()`.
- Multi-line strings via `|-` (block scalar, strip trailing newline).

### Calling localized strings from YAML

Inside any expression position or interpolation token, use the `$t`
builtin:

```yaml
- assign:
    greeting: |-
      $input.greeting != null && $input.greeting != ""
        ? $input.greeting
        : ($summary.first_name
            ? $t("home.greeting_with_name", { first_name: $summary.first_name })
            : $t("home.greeting_default"))

- render:
    type: list
    header: '{$t("home.card_header")}'
    body: |-
      {$greeting}

      {$t("home.status_listings", { count: $summary.listings_count })}
    footer: '{$t("home.card_footer")}'
```

`$t(key, vars?)`:
- `key` format: `<flow_id>.<string_key>` — both required.
- `vars` is an optional object literal: `{ count: $n, name: $first_name }`.
  Object expressions in the DSL support shorthand and arbitrary
  expressions as values.
- Resolution order: requested locale → `DEFAULT_LOCALE` (`'ar'`) → return
  the key with a console warning. **Always create the default-locale
  file**; missing keys there will leak into the UI.
- Namespaces are **pack-scoped**: `$t` resolves only against the calling
  pack's own locale files, so a flow id used by another pack too (e.g.
  `home`) is fine — you can't reference (or collide with) another pack's
  keys.

### Locale source

Locale is read from `user.locale` (per-tenant) and injected as
`$locale` into flow vars. If absent, defaults to `ar`. The `$t` builtin
closure-captures the resolved locale on `runFlow`/`resumeFlow`, so
expressions don't pass it explicitly.

### What stays inline

These don't go through `$t`:

- `flow_id`, `version`, `description` — flow metadata, not user copy.
- Platform-default error / suspend copy (registered by the pack via
  `PlatformLocaleRegistry`) — pack-supplied at boot, currently Arabic-only.
  Future work will route these through `$t` too.
- Field `describe` strings in `input:` — these become Mastra tool
  descriptions surfaced to the LLM, not the end user.

### Adding a new locale

1. Drop `<flow>.locale.<lang>.yaml` next to the flow.
2. Translate every key from `<flow>.locale.ar.yaml`.
3. Set `user.locale = '<lang>'` on the user/tenant config.
4. The runtime hot-loads the locale at boot; restart `mastra dev` to
   pick it up.

If a key is missing in the new locale, the runtime falls back to `ar`.

## §11b Platform builtin primitives

Three primitives are available to every flow without any pack-side
registration. They live in `src/platform/primitives/<name>/primitive.ts`
and are auto-discovered + injected into every FlowTool's primitives
map by `src/platform/pack/tools.ts`. Pack-supplied primitives shadow
them on name conflict (intentional override path for tests).

Machine-readable catalog: `dist/octwin-platform-kb/platform-primitives.json`
(regenerated on every `npm run build`).

### `do: notify` — channel-agnostic cross-user delivery

Sends a UiIntent to a DIFFERENT user (not the inbound sender). Resolves
the recipient contact, selects the per-channel strategy (WhatsApp push /
web SSE / future channels), persists a `notifications` row, returns
`delivered` / `pending` / `failed`. Pre-turn drain hook replays pending
notifications on the recipient's next inbound.

```yaml
- do: notify
  args:
    to:
      contact_id: '$lead.seller.contact_id'         # Form A
      # OR
      channel:                whatsapp              # Form B
      channel_contact_handle: '$lead.seller.channel_contact_handle'
    intent:
      render_intent: detail_card
      header: '{$t("…")}'
      body:   '{$t("…")}'
      buttons:
        - { title: 'قبول', on_select: { invoke: leads, with: { action: send_message, kind: accept } } }
    memory_note:   'Pending offer from buyer'
    visibility:    permanent                        # permanent | ephemeral | diagnostic
    echo_delivery: true
```

Full reference: [docs/18-platform-notify.md](../../../../docs/18-platform-notify.md).

### `do: send_message` — same-conversation outbound (mid-flow)

Emit an additional message OUTSIDE the normal `render:` path. Use when a
SINGLE step needs to emit multiple messages — e.g. a "searching now…"
progress message before the actual search render.

```yaml
- do: send_message
  args:
    to:   '$conversation.channel_contact_handle'
    text: 'لحظة من فضلك بأبحث...'
    # OR
    # intent: { render_intent: text_card, body: '...' }
  outputs:
    delivered: []
    failed:    []
```

Distinct from `notify` (same-conversation, no persistence beyond the
adapter's outgoing-capture trail).

### Memory notes via `note:` (unified list)

Leave a breadcrumb the agent reads on its next turn without rendering
anything user-visible. (Lineage: the retired `inject_memory_note`
primitive became `emit_envelope.memory_note`, now narrowed + renamed to
the `note:` node.) A run accumulates ONE `memory_notes` list — `note:`
nodes and renders' `memory_note:` sugar both push onto it.

```yaml
# Use the native flow-runtime node — no primitive needed.
- note: 'User accepted the offer — congratulate them next turn.'
- end: applied
```

**Delivery depends on the flow kind:**
- **Tool flows** — the run's LAST note surfaces on the ToolResponse's
  `memory_note` (derived by the envelope-mapper) and reaches the agent
  next turn as a `<system-reminder type="tool_action">` note.
- **Pre-turn hooks + media handlers** — the WHOLE list feeds the
  `TurnDataStore` briefing queue; `PreTurnBriefingProcessor` drains it
  into the LLM's input messageList at the next `agent.generate` call as
  `<system-reminder type="pre_turn_briefing" source="flow">…</…>` signals.
- **Post-turn flows** — no note channel (the store is per-request); see
  `docs/BACKLOG.md`. (An earlier claim here that EVERY flow's notes drain
  as pre-turn briefings was wrong — tool-flow notes used to be dropped.)

The note's text supports the same `{$expr}` interpolation every other
flow string uses. A render's `memory_note:` differs only in firing
together with a card; `note:` is the "breadcrumb, no card" path.

### Naming convention

Platform builtin primitives use **short generic verb names** (`notify`,
`send_message`, `get_media`; future candidates: `schedule_at`,
`audit_log`). No `platform_` prefix — packs that want to shadow a
builtin do so with the same name (intentional override). Folder name
MUST match the primitive's declared `name` (enforced by the discovery
loader; throws at boot on mismatch).

## §11c Platform media service

Phase 11 added project-scoped media handling — storage upload, ref
minting, last-mile URL resolution — to the platform. Two flow
primitives expose the read/write surface so YAML composes media work
without crossing TS boundaries:

### `do: get_media` — resolve a media asset by id or ref

```yaml
- do: get_media
  args: { media_id: '$input.image_media_id' }
  bind: m
  outputs:
    found:
      # $m has: { media_id, kind, mime_type, caption, url, metadata }
      - do: ...
        args: { url: '$m.url', mime_type: '$m.mime_type' }
    not_found: []
```

Accepts a `MEDIA-XXXXXXXX` ref OR a raw UUID. The returned `url` is a
fresh public/signed URL the channel adapter can consume; the renderer
already does last-mile MEDIA-ref expansion automatically when a
`image_url:` field carries a ref instead of a URL.

### `do: set_media_metadata` — write JSONB patch onto a media asset

```yaml
- do: set_media_metadata
  args:
    media_id: '$input.media_id'
    patch:    { analysis: '$r.analysis' }
  outputs:
    ok:     []
    failed: []
```

Top-level keys in `patch` replace existing keys wholesale (jsonb `||`
semantics). Pack media-handler flows write vision/OCR results here so
later tools can read them via `get_media`.

### Inbound media flow (platform-side)

Every inbound image / document the channel adapter delivers triggers:

  1. `uploadMedia(...)` — bytes go to Supabase Storage; a `media_assets`
     row gets a `MEDIA-XXXXXXXX` ref minted from its UUID.
  2. `runMediaHandlerFlow(...)` — if the pack manifest declares
     `media_handlers.image:` / `media_handlers.document:`, that flow
     runs with `{ media_id, media_ref, mime_type, caption, kind }` as
     input. Pack-side YAML enrichment (vision analysis, OCR, memory
     notes for the next turn) lives here.
  3. The agent sees a tiny JSON envelope:
     `{ "type": "image", "media_id": "MEDIA-…", "mime_type": "…", "caption": "…" }`
     — never a URL, never base64.

Pack flows that accept media in their inputs (e.g. `publish-listing`'s
`photos: array<string>`) take **media_id refs**, not URLs. The
renderer's pre-resolution pass swaps refs for real URLs before any
channel adapter sees them.

### Marking a flow as platform-only

The media-handler flow is invoked by the platform — not the agent —
so it's declared with `internal: true` at the top of the YAML:

```yaml
flow_id: analyze-media
version: 1
internal: true        # platform-only; never appears in any agent tool list
```

`buildPackTools` throws at boot if any `agents.<id>.tools:` list
references an internal flow. The flow is still registered in the
per-pack FlowTool registry (`PackRegistry.getFlowRegistry(packId)`)
so platform-internal callers (`runMediaHandlerFlow`, future cron /
webhook / on-publish triggers) can invoke it by id.

Use `internal: true` for:
  - Media handlers (this section).
  - Cron / scheduled jobs (future).
  - Webhook receivers triggered by external systems (future).
  - Sub-flows that exist only for `use:` / `goto:` composition.

## §12 Don't

- ❌ Don't use `${...}` for interpolation — the runtime parses `{...}`.
  `${$x}` becomes the literal `$<value-of-x>`.
- ❌ Don't put `[0]` etc. unquoted in YAML — `'$x[0]'` is fine; `$x[0]`
  is a YAML flow-sequence syntax error.
- ❌ Don't use bare side-effect imports for primitives modules — they
  get tree-shaken in production. Always use named imports + use the
  exported value transitively.
- ❌ Don't fall through a step body. The last node must be `goto`,
  `end`, or `suspend`.
- ❌ Don't suspend twice in one node sequence without an intervening
  `goto` and on_resume — only one outstanding suspend per run.

---

## Auto-generated JSON Schema (for tooling)

This block is regenerated by `npm run schema:context`. Keep the prose
above hand-written; this block stays mechanical.

<!-- BEGIN_GENERATED_SCHEMA -->
<!-- Run `npx tsx scripts/generate-schema-context.ts` to refresh. -->

```json
{
  "$ref": "#/definitions/FlowDef",
  "definitions": {
    "FlowDef": {}
  },
  "$schema": "http://json-schema.org/draft-07/schema#"
}
```
<!-- END_GENERATED_SCHEMA -->
