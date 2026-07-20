# 13a — YAML DSL Reference

**Status:** AUTHORITATIVE. Source files cited here are the canonical truth; this doc summarises the contract for pack authors so the IDE / autocomplete / lint can match.

**Pair with:** [12-pack-authoring.md](12-pack-authoring.md) (conventions, conditional values, runtime ergonomics) · [16-platform-stdlib-templates.md](16-platform-stdlib-templates.md) (stdlib templates) · [20-two-conversation-model.md](20-two-conversation-model.md) (envelope + dispatch contract) · the **Flow-DSL Node Handbook** Artifact (author tutorial, shipped-pack examples): <https://claude.ai/code/artifact/fc2e730a-549a-4c3f-8656-1be306a6b5a4> (source [artifacts/flow-dsl-nodes-guide.html](artifacts/flow-dsl-nodes-guide.html)).

---

## §1 Step grammar (14 op-keys)

A node has **exactly one** op-key (zero or more than one = schema error). Source: [`schemas/nodes.ts`](../src/platform/flow-runtime/schemas/nodes.ts).

```
do  |  if  |  branch  |  dispatch  |  collect  |  pause  |  render  |  end  |  assign  |  note  |  answer  |  goto  |  foreach  |  track
```

### `do` — call a primitive

```yaml
- do: get_property
  args: { id: '$resolved_id' }                  # optional
  bind: property                                # optional; defaults to '$get_property' (Phase 1 Win 1.3)
  outputs:                                      # optional; for multi-output primitives
    not_found: []                               # empty = canonical handler (see §4)
    found:
      - render: { ... }
      - end: applied
```

- `bind:` defaults to the primitive name. E.g. `do: get_property` → `$get_property`.
- **Port-returning primitives bind the OK-port PAYLOAD directly.** `$<bind>` IS the
  payload — there is no `{ output, data }` wrapper. For `db_*` that means
  `do: db_rpc … bind: hits` → `$hits` is the rows array (or the scalar / single row);
  read `$length($hits)`, never `$hits.data.rows`. A **failure port** (`failed` / `fail`
  / `error`) on a bare `bind:` **throws loudly** (no more silently binding a truthy
  `{reason}` — the old "no results" footgun). To handle a failure in-flow, declare it
  as an explicit `outputs:` port instead.
- Empty-array `outputs: { <port>: [] }` triggers the canonical handler from the port-defaults table (§4).
- **No-result is a PORT, not a follow-up `if`.** `db_select` and `db_rpc` emit an
  `empty` port when the query returns no rows (`[]`) or a null single row — so route
  `outputs: { ok: [...], empty: [...] }` instead of binding then writing
  `if: $length($x) == 0` / `if: $x == null`. (A bare `bind:` still binds the payload
  — `[]`/`null` — regardless, so callers that don't care are unchanged.) `db_insert` /
  `db_update` / `db_upsert` / `db_delete` have **no** `empty` port.

**When to route via `outputs:` vs keep a follow-up `if`** — the rule the pack follows:

| Use `outputs:` (declarative port routing) | Keep a bare `bind:` + `if` |
|---|---|
| **Both branches terminate** (`goto` / `end` / `pause`) — e.g. lookup → `ok: [goto found]`, `empty: [goto not_found]`. | **Guard-clause with a long ok-path** — `if (empty) → render+end`, then the main flow continues. Early-return reads better than nesting the whole flow under `ok:`. |
| The outcome is the FRESH bind being empty/failed. | **Fall-through chain** — each miss continues to the next step. An `outputs:` body MUST terminate; it can't fall through (`empty: []` is rejected by the loader). |
| | **Compound condition** (`$x == null \|\| not_owner`), or a check on a **derived / sub-field / accumulator** var rather than the fresh bind. |

**Errors vs. empties — let the platform net handle the unexpected; keep guards for the expected.**
An *uncaught throw* (a thrown exception, or a `failed` port on a bare `bind:`) is caught by the platform's agent error-net (`SyntheticDispatchInjector`): the agent receives a `{ success: false, instructions, data: { error } }` envelope, produces a **localized, context-aware** apology that *translates* `data.error` into a plain-language correction nudge, and an operator `platform_error` chip is written. So an unexpected failure already gets graceful, guiding UX **for free** — better than any generic card you'd hand-roll, plus ops visibility. Author with that in mind:

- **Unexpected failure / "this shouldn't happen"** (an op that errored, an invariant that should always hold) → **let it throw.** Do NOT wrap it in `outputs: { failed: [ …generic error card… ] }`. The net beats the card. (The raw error goes to the LOGS + the operator chip; the agent gets a genericised apology — it can't fix a DB crash by changing its input.)
- **Agent-correctable failure** (a missing/invalid/malformed arg the agent CAN fix — required field, bad format, out-of-range, unknown SKU) → return the **`invalid`** port (`outputs: { invalid: [] }`) with a specific `reason`. The canonical handler hands that reason to the agent as `instructions` with **no render** (agent-protocol §1), so the agent corrects its tool input and *retries* — the loop is bounded by `MAX_AGENT_STEPS`. This is the "error details guide the agent" path: unlike `failed` (a terminal card) or a throw (genericised), the specific detail reaches the model. Keep the `reason` a clean, agent-facing instruction — never a raw system/SQL string (those are class-1 throws → logs only).
- **Expected empty / not-found / unauthorized** → **keep the `empty` port / guard.** These are normal *domain* outcomes, not errors — they deserve a tailored, non-alarming reply + a next step (offer search, a picker, "you don't own this listing"). Do NOT route them through the net: it frames everything as a *system error* ("sorry, a technical error…"), which is the wrong tone for a normal not-found, and degrades UX.
- **The throw-vs-empty trap (decisive):** the net only fires on a **throw**. `db_select`/`db_rpc` return the `empty` port (or bind null) on no-rows — they do **not** throw — so a guard like `if $x == null → …` after a lookup is **load-bearing**: deleting it does NOT reach the net, the flow just sails on with null into a wrong result. Only the `failed` port throws.
- **Flow-local failure handling earns its keep ONLY when it does more than display an error:** recovery (retry / fallback / alternative route), resume-preserving UX (keep the workflow run alive so the user fixes one field and *resumes* — the net **ends** the run), pre-validation that blocks **partial/destructive** work (e.g. a contact-leak blocker before a relay send), or a structured card with domain actions. A `failed:` port whose body just renders a generic error is redundant — delete it and let the throw reach the net.
- **`detach: true`** — fire-and-forget: kick the primitive off WITHOUT awaiting and continue the
  flow immediately. For best-effort side effects (analytics touches, non-critical writes) that must
  not add turn latency. The detached promise keeps the tenant/pack ALS context, so a scoped `db_*`
  write still targets the right install. Errors are logged, never surfaced. No result → mutually
  exclusive with `bind:` / `outputs:`. (A null result is handled with `bind` + `if: '$<bind> == null'`
  — the old `on_null:` sibling was retired.)

**Generic data/IO primitives** (platform-shipped, install-schema-scoped — flows need no pack repo):
`db_select` / `db_insert` / `db_update` / `db_delete` (`where` required) / `db_upsert` for
single-table CRUD; `db_rpc` to call a **migration-defined** Postgres function (joins / aggregations /
fuzzy matching); `storage_remove` for deleting install-bucket blobs. Structured
filters only (no raw SQL). Full schemas: [`docs/16b`](16b-platform-stdlib-primitives.md).

### `if` — conditional branch

```yaml
- if: '$user.role == "seller"'
  then:
    - assign: { greeting: 'Hello seller' }
  else:
    - assign: { greeting: 'Hello buyer' }

# `{ if, goto }` sugar — a conditional DIRECT JUMP (≡ then: [{ goto: … }]).
# `goto` XOR `then`; `else:` (a body) may still ride alongside. Mirrors the
# `entry:` grammar's `{ if, goto }` shape.
- if: '$seller == null'
  goto: render_no_match
```

Both `then`/`else` branches optional. Unmatched → continue (unless `else:` is present). `{ if, goto }` and `{ if, then }` are mutually exclusive — the goto IS the then-branch.

### `branch` — multi-way dispatch

```yaml
- branch:
    - when: '$count == 0'
      then: [ ... ]
    - when: '$count == 1'
      then: [ ... ]
    - default: true
      then: [ ... ]
```

Each case must have exactly one of `when:` or `default: true`.

### `dispatch` — switch by key

```yaml
- dispatch:
    key: '$input.action'              # evaluated, stringified, matched against `cases`
    cases:
      contact: [ ... ]
      offer:   [ ... ]
      cancel:  [ ... ]
    default:   [ ... ]                # optional; runs on no match (else continue)
```

Sugar over a `branch:` of equality checks on **one** key — use it for discriminated enums
(action / status / type). Use `branch:` when cases test different keys or non-equality
conditions (ranges, null-checks on different vars). Terminal iff every case (and `default`,
if present) terminates — same leniency as `branch:`.

### `collect` — gap-collection loop (the engine owns suspend/resume)

```yaml
- collect:                            # accumulator is `$state` (the universal turn-merge folds this
                                      #   turn's `$input` delta in before any node runs)
    derive:                           # computed fields merged into $state each entry, BEFORE
      remaining_egp: '<expr>'         #   before_each + the missing-check, in order (each sees the
      total_egp:     '<expr>'         #   prior). Declarative form of `assign: { state: $merge($state, ...) }`.
    transient: [_scratch, run_id]     # $state keys that never persist → exposed as `$collect_transient`
    before_each:                      # re-derivation / validation, run at the TOP of every loop
      - assign: { ... }               #   iteration (sees auto-resolve merges same turn); may `goto`
    before_prompt:                    # runs AFTER the missing/validate pass (sees $collect_missing
      - assign: { view: '...' }       #   + $collect_errors), before complete-or-prompt; build the
                                      #   progress view here, not in before_each (too early to see them)
    gap_instructions: '<expr>'        # instructions for the ENGINE-SYNTHESIZED gap prompt (any
                                      #   field with no `prompt:` — the engine renders a field_prompt
                                      #   for $collect_field + data:{status:gap} carrying these)
    fields:                           # checked IN ORDER; first eligible+missing one is prompted, then suspends
      - field: type                   # FIXED picker — declarative options (engine desugars to a
        options:                      #   field_prompt list at load; no hand-built prompt/ps_sections)
          header: pack.x.header; body: pack.x.body; cta: pack.x.cta   # locale keys (wrapped in $t)
          sections:                   #   OR a single section: `enum: <name>` + `values: [...]`
            - { title: pack.x.sec, enum: types, values: [apartment, villa] }   # $enum_label-labelled
            - { items: [ { value: "1", label: pack.x.one } ] }                 # explicit-label
      - field: city                   # DYNAMIC picker — fetch rows + map to options
        options:                      #   (zero rows → no render → synthesized gap prompt)
          header: pack.x.h; body: pack.x.b; cta: pack.x.c
          fetch: { do: db_select, args: { table: areas, where: { is_active: true } } }   # array → rows defaults to the bind
          map:   { value: '$item.name_en', label: '$item.name_ar' }   # selection: { city: $item.name_en } is ALWAYS backfilled — omit it
      - field: compound               # CASCADE picker — narrow via a coarser rung only when over cap
        options:
          header: pack.x.h
          body:   '$t("pack.x.b", { area: $coalesce($ci.area_name_ar, $state.city) })'   # expr body; $ci in scope
          cta:    pack.x.c
          fetch:  { do: db_rpc, args: { fn: compounds_in_area, params: { p_city: '$state.city', p_developer_id: '$state._developer_filter' } }, bind: ci }
          rows:   '$ci.compounds'     # select the array off the envelope
          map:    { value: '$item.id', label: '$coalesce($item.name_ar, $item.name_en)', selection: { compound: '$coalesce($item.name_ar, $item.name_en)', compound_id: '$item.id' } }   # `compound` OVERRIDES the backfill (stores the name, not value); compound_id is an extra key
          auto_resolve: true          # 1 compound → auto-pick, no tap
          narrow:                     # shown only when compounds > $caps.list.maxRows (platform-injected)
            field: _developer_filter
            options:
              header: pack.x.dh; body: pack.x.db; cta: pack.x.dc
              fetch: { do: db_rpc, args: { fn: resolve_developer, params: { p_city: '$state.city' } }, bind: dv }
              rows:  '$dv.candidates'
              map:   { value: '$item.id', label: '$item.name' }   # _developer_filter backfilled; add only reset keys e.g. selection: { compound: null }
              auto_resolve: true       # single-developer city → auto-pick → compound re-fetches filtered
      - field: compound
        when: '$state.city != null'               # eligibility (cascade rung); default: always
        prompt: [ ... ]                           # DB-driven picker — keeps an explicit prompt
      - field: area_m2                            # NO prompt → engine-synthesized gap prompt (free-text)
        label: '$t("pack.x.area_m2")'             # human label → exposed in $collect_missing_labels (else field name)
      - field: bedrooms                           # ENRICHMENT — never blocks complete AND never prompted;
        required: false                           #   listed collectively via $collect_optional_pending(_labels)
        label: '$t("pack.x.bedrooms")'
      - field: seller
        validate: { when: '<expr>', error: '<expr>' }   # PRESENT-but-invalid → block complete + re-prompt
      - field: license_photo                            # UPLOAD field — a 📷/📎 gap, no picker
        type: media                                     #   folds a MEDIA- ref from $inbound_media
        media_kind: image                               #   image | document (required on media fields)
        # multiple: true                                #   optional → array of refs
        label: '$t("pack.x.license_photo")'
        on_collect:                                     # DERIVATION HOOK — runs ONCE when this field first
          - do: analyze_image                           #   fills, in the seam AFTER the fill / BEFORE the
            args: { image: '$state.license_photo', json: true, prompt: '...' }   #   next-field pick, so
            bind: ocr                                   #   fields it assigns are satisfied + SKIPPED this
            outputs:                                    #   turn (e.g. OCR a photo → prefill name/license_no).
              ok:     [ { if: '$ocr.json != null', then: [ { assign: { 'state.name': '$state.name ?? $ocr.json.name' } } ] } ]
              failed: []                                # HOOK MODE: empty port = no-op continue (never ends the flow)
    complete: [ { goto: approve } ]   # runs when no REQUIRED field is missing AND optionals are offered/done
```

The engine loop: this turn's `$input` delta has ALREADY been folded into `$state` (the accumulator)
by the universal turn-merge — collect no longer merges `$input` itself, so a stale prior-turn delta
can never re-clobber `$state`. THEN, at the top of EVERY iteration: apply
**`derive`** (computed $state fields, in declaration order) → run `before_each` → compute the
missing/validate sets (`$collect_missing` / `$collect_errors`) → run **`before_prompt`** (build view
state from those sets) → find the first field where `when` (default true) passes AND it's **missing**
(default `$state.<field> == null`) OR its **`validate.when`** fires → run its `prompt` (or, when the
field declares none — or a `validate` failed, **or the prompt produced no render** — the
**engine-synthesized** gap prompt) and **suspend**; on resume the step re-enters and re-checks; when none
missing, run `complete`. **`derive` + `before_each` re-run every iteration** (not once before the
loop), so an `auto_resolve` merge mid-loop is followed by fresh re-derivation/validation that sees the
merged bag THIS turn (e.g. resolve a just-auto-resolved compound → its name/developer) rather than one
turn late; a flow with no auto-resolve loops once, so they still run exactly once per entry. A
`prompt`/`before_each`/`before_prompt` that returns a non-`continue` outcome (`goto` to a delegated
render/disambiguation step, `end`, …) short-circuits. **`before_prompt`** runs after the missing pass
on every entry (so it sees `$collect_missing`, unlike `before_each`) and its vars reach both the
prompt's `progress` and a downstream `complete` step.

**Per-field `on_collect`** is the derivation hook for "when THIS field fills, compute others." It runs
**once**, the first time the field becomes non-empty (filled by an agent arg, media auto-adopt, or a
prior field's `on_collect`), positioned in the seam **after** the fill but **before** the missing pass —
so fields it assigns into `$state` are seen and **skipped that same turn** (e.g. a license-photo field
whose `on_collect` OCRs the image and prefills `name`/`license_no`). Once-per-field is tracked in
`$state._collect_derived` (persists across suspends, like `_collect_skipped`). All three hooks
(`on_collect`/`before_each`/`before_prompt`) run in **hook mode**: a hook enriches `$state`, it never
renders or terminates, so a `do:`+`outputs:` port left **empty** (`failed: []`) is a no-op `continue` —
NOT the canonical render+end — and no port needs a terminal `goto`/`end`. (An explicit `goto`/`end` in
a hook still short-circuits, as above.) The **no-render fallback** lets a picker whose
option fetch came back empty simply *not render* (an `if options>0` with no `else`) → the engine
shows the synthesized gap prompt instead of an empty card (previously a silent suspend when no
`default_prompt` was declared). **Cascades** are ordered
`when:`-gated rungs (each prompt reads prior `$<into>.*`); the engine exposes the missing
field-name list as **`$collect_missing`** (required-missing only — optionals don't gate completion),
those fields' resolved `label`s in order as **`$collect_missing_labels`** (a progress card reads this
instead of hand-mapping keys → labels; falls back to the field name), the progress counter as
**`$collect_total`** / **`$collect_done`** / **`$collect_remaining`** (over required fields), per-field
validation errors as **`$collect_errors`**, the scratch-key list as **`$collect_transient`** (the
node's `transient:` plus the engine's own `_collect_*` bookkeeping keys, so a persist site's
`$omit($state, $collect_transient)` never leaks engine internals), and the
currently-prompted field as **`$collect_field`** for a progress/preview render. A field with
**`required: false`** is an enrichment field — it **never blocks `complete` AND is never prompted**
(no per-field offering loop); it's collected if the caller supplies it, and while still empty surfaces
as **`$collect_optional_pending`** (+ resolved labels **`$collect_optional_pending_labels`**) so a
confirmation/preview card can list them COLLECTIVELY ("you can also add: …") rather than looping the
seller through each one. A field with no `prompt:` gets the **engine-synthesized gap prompt** —
a `field_prompt` for `$collect_field` carrying `gap_instructions` + `data:{status:gap}` (a single
bare `{ field: x }` routes to it). **`options:`** is authoring sugar for a picker — desugared at load into the canonical
`prompt: [ (fetch?) assign ps_sections + fp_*, use: field_prompt_render ]`. `field_prompt_render` is a **platform-provided**
template (`src/platform/flow-runtime/templates/`) — you never author it, and the sugar works in any pack out of the box; a pack
may ship its own `templates/field_prompt_render.template.yaml` to override the render. **Static**: `enum`+`values`
/ `items` / `sections`. **Dynamic**: `fetch: { do, args, bind? }` (run a `do`, bind its result —
default `__opt_<field>`) + `rows:` (the array off the bind — default `$<bind>` for `db_select`;
envelope RPCs select a sub-path, e.g. `rows: '$ci.compounds'`) + `map: { value, label, selection,
description? }` shaping each row into an option; zero rows → no render → synthesized gap prompt.
`header`/`body`/`cta`/`footer` take a bare locale key (→ `$t(key)`) **or** a full `$…` expr verbatim (e.g.
`'$t("k", { area: $state.city })'`; the `rows` bind is in scope, so a body may read
`$ci.area_name_ar`). All four are optional — an omitted one simply isn't rendered. **`auto_resolve: true`** — a picker whose fetch yields exactly ONE option
merges that lone selection into the bag and renders nothing; the collect engine's progress-loop
re-runs the missing pass (no pointless one-option tap). **`narrow: { field, options }`** — a CASCADE:
a coarser picker shown first, but ONLY when the target list won't fit one render. The fit-check is
the **platform-injected `$caps.list.maxRows`** (a channel-aware decision → the platform owns it; the
pack YAML stays channel-blind). Once the narrow field is set (its pick resumes → collect re-enters)
the target re-fetches filtered and now fits; the narrow rung may itself `auto_resolve`. (A field
whose prompt has branching the sugar can't express keeps an explicit `prompt:`.) Every field renders
through the `field_prompt` intent (§2): with options → a
picker, without → a text ask. Reuses the existing pause/resume + render machinery (no new
persistence). Replaces the hand-rolled `collect→gap_analysis→field_picker_*→pause→goto collect` machine.

**`type: media`** — an UPLOAD field (never a picker; `options:`/`prompt:` are forbidden, `media_kind:
image|document` is required). Its synthesized gap is a `field_prompt` carrying `expect: 'media'` +
`media_kind` (a 📷/📎 ask; a web adapter can draw an attach affordance, WhatsApp just receives the
file). On the next turn the engine **deterministically adopts** the first unconsumed
`$inbound_media` ref of the matching kind into `$state.<field>` — independent of whether/how the agent
named it (so the fold never depends on the agent re-invoking with the right arg; see agent-protocol
§Media). A wrong-kind upload is **not** adopted: it sets `$collect_errors.<field>` and re-prompts with
a ⚠️. `multiple: true` accepts an array of refs (completion needs ≥1 when required). An **optional**
media field (`required: false`) is not silent enrichment like an optional text field — it is OFFERED
with a **Skip** button (`skip_title:` sets the label; a Skip tap resumes with `$control.skip=<field>`),
deferred until all required fields are satisfied, and blocks completion until the user uploads OR skips.
Downstream `do:` nodes read the ref from `$state.<field>` like any other collected value. `$inbound_media`
(`[{ ref, kind }]`) is bound fresh per turn (never restored from a suspend snapshot) — see
[`docs/19-platform-media.md`](19-platform-media.md).

### `render` — emit a UI card

```yaml
- render:
    render_intent: list_picker                  # see §2 for the 9 intents
    body: '...'
    # intent-specific fields...
  memory_note: '...'                            # optional; curated breadcrumb (Phase 15 flat envelope)
  data: { ... }                                 # optional; merged into ToolResponse data
  instructions: '...'                           # optional; agent-facing instructions for render-less completion paths
```

Only these three sibling fields (`memory_note`, `data`, `instructions`) are valid on a `render:` node — the schema is now `.strict()`, so a typo'd sibling is a load-time error, not a silent no-op. The legacy `expose:`, `narrate:`, `memory_injection:`, `agent_instructions:`, `error_messages:`, `message_ar:`, `path:`, `footer:` (top-level), `header:` (top-level), and `buttons:` (top-level) fields are all retired — they either moved inside the `render_intent:`-keyed body or were removed (the flat envelope; fallback copy now lives in the platform locale, not per-render).

`memory_note:` on a render is the Phase-15 way to attach a one-turn breadcrumb. Presence IS the opt-in — no separate `persist_memory` flag.

### `pause` — terminator marker for "wait for user input"

```yaml
- render:                                       # the render that goes WITH the pause
    render_intent: detail_card
    body: '...'
    buttons: [ { title: '...', on_select: { resume: '$self', control: { approved: true } } } ]
  memory_note: '...'                            # optional; on the render, not the pause
- pause:                                        # terminator — body holds ONLY resume mechanics
    on_resume:                                  # required when the flow resumes
      - if: '$control.approved == true'          # control flag (see on_select.control)
        goto: apply                             # {if,goto} sugar — direct jump
      - goto: aborted
```

`pause:` is a marker, not a render container. The card that the user sees comes from the `render:` node directly above. The pause body holds ONLY:

| Field | Purpose |
|---|---|
| `on_resume` | NodeOrSeq run when the channel delivers a resume payload |

A next-turn breadcrumb rides the **preceding render's `memory_note:`** (the pause itself carries none — the old `memory_note_template` was retired).

Resume payload — two channels (a tap's `on_select` chooses per key):
- **`$input` (DATA, this-turn)** — `on_select.with: { foo: bar }` becomes THIS turn's `$input` (so `$input.foo` is usable in `on_resume`), and the universal turn-merge folds it into `$state` / the collect accumulator. `$input` is per-turn (not carried across turns); read `$state.foo` for a value that must survive a suspend. Use `with:` for field-updates a picker contributes. (The retired `$resume` channel was the pre-fold staging of this `$input`.)
- **`$control.*` (CONTROL)** — `on_select.control: { approved: true }` routes to the reserved `$control` namespace, read in `on_resume` (`$control.approved`). It is **NEVER** merged into `$input` or the collect bag — so approve/decline/retry flags can't pollute persisted data. Use `control:` for decisions (approve/abort/retry), `with:` for data.

### `end` — terminate the flow

```yaml
- end: applied                                  # 'applied' | 'aborted' | 'failed'
  publish:                                      # optional; publish sugar (renamed from `with:`)
    image_type: 'floorplan'                     # publishes to $in_turn.* (or $pre_turn / $post_turn)
    cached: 3600                                # meta-key: opt the publish into the cross-turn cache (seconds)

# Object form — render a text_card with `body`, THEN terminate. Collapses the
# ubiquitous `render: { render_intent: text_card, body } ; end: <status>` pair:
- end: { status: failed, body: '{$t("leads.errors.lead_not_found")}' }

# Object form with `instructions` — set the envelope brief WITHOUT a render, so
# the AGENT is not suppressed and composes/acts on it (agent-protocol §1). This
# is the "agent-correctable failure" shape (backs the canonical `invalid` port):
- end: { status: failed, instructions: '$v.reason' }   # e.g. "field X is required"
```

Only three terminal kinds: `applied`, `aborted`, `failed`. The legacy `gap` terminal kind is retired — flows that need a "suspended" outcome use a `pause:` step.

**Pick the terminal that tells the truth — a failure branch ends `failed`/`aborted`, NOT `applied`.** The terminal drives the whole envelope: `end: applied` → `success: true` + `data.status: 'applied'` + a default "…applied" memory note; `end: failed` → `success: false` + `status: 'failed'` + "…completed without applying". A rendered failure branch delivers its card AND halts the agent the same either way (the suppression halt is success-independent — [`docs/20`](20-two-conversation-model.md) §Suppression check), so there is **no reason to reach for `end: applied` on a failure path**. Doing so lies to thread memory + analytics (it once made an add-to-cart failure report success). The loader **rejects** a negative port (`failed`/`fail`/`error`/`aborted`) that renders a card yet ends `applied`. (A negative port with NO render that ends `applied` — a silent graceful-degrade where the flow outcome genuinely applied — stays legal.)

**`end: { status, body }`** is exactly equivalent to a `render: { render_intent: text_card, body }` node immediately followed by `end: <status>` — the interpreter runs the same render path, so `body` supports `{$expr}` interpolation like any render body. Use it for the common terminal-message case (errors, not-found, empty results); keep the explicit `render:` node when the render needs a `memory_note`, `data`, a non-`text_card` intent, or a `body: |` block (the object form carries only a status + body). The `publish:` sibling still works alongside it (`end: { status: applied, body: '...' } ` + `publish: {...}`).

`publish:` is sugar for publishing lifecycle **vars** at the moment the flow ends (renamed from `with:` — the old name read like "return data"; it never touched the envelope). Sibling to `assign:`. The `cached:` meta-key (number, seconds) opts the publish into the cross-turn `TurnDataStore` cache. Other keys are resolved via `resolveValue` (string-is-`$expr`).

### `assign` — bind expressions / structured values to vars

```yaml
- assign:
    greeting:       '$t("home.greeting")'   # expression (references a $var)
    count:          '$length($properties)'
    summary:        '"Found " + $count + " units"'  # string-leading expression (has $)
    effective_city: null                    # YAML null — no quoting
    is_direct:      false                    # YAML bool
    field:          type                     # plain literal string "type"
    result:         { data: '$rows' }        # YAML object; nested $expr resolves
    ids:            ['$a', '$b']             # YAML array
```

Each value is resolved via `resolveAssignValue` (same recursive `$expr` handling as `end: publish:`). Resolution rules for a **string** value:

- first non-ws char is `$` → **expression** (`$x`, `$x ? "y" : "z"`). A `$`-leading expression may itself contain `{$…}` string literals (`$lines(['🏠 {$ref}'])`); those interpolate **per-literal** during evaluation, so this case is checked **before** the `{$…}` rule below — otherwise the whole expression would be mistaken for a top-level template.
- first non-ws char is `{`/`[` (but not a bare `{$…}` hole) → object/array **expression** (`'[1, 2, 3]'`, `{ data: $rows }`)
- contains `{$…}` → **interpolated** top-level template (`'id={$user.id}'`)
- contains `$` (anywhere else) → **expression** (`"a" + $b` — a literal-leading concatenation)
- otherwise → **plain literal string** (`type`, `manage`)

Non-string YAML (object / array / `null` / bool / number) is resolved recursively. So `effective_city: null` and `result: { data: $rows }` "just work" — **no `'"x"'` / `'null'` / `'true'` quoting ritual** (the `assign-quoted-literal` lint flags leftovers). Result is bound into the flow's vars map (accessible as `$<key>` downstream).

### `note` — mid-flow memory-note breadcrumb

```yaml
- note: 'Seller uploaded {$length($input.media)} photo(s)'   # $expr / {$expr} template both work
```

Push ONE breadcrumb onto the run's `memory_notes` list without rendering a card — that is its whole job. Non-terminal (the flow continues). Formerly `emit_envelope`, which also carried `data`/`instructions` — that overlap was retired: use **`answer:`** for "data + brief, the agent composes" and **`render:`** for card-bearing envelopes.

**Delivery depends on the flow kind.** In a **tool flow** the run's LAST note (from `note:` or a render's `memory_note:` — one unified list) surfaces on the ToolResponse and reaches the agent next turn as a `tool_action` system note. In a **pre-turn hook / media-handler flow** the whole list drains via the TurnDataStore briefing queue as `pre_turn_briefing` signals. **Post-turn flows have no note channel** (see `docs/BACKLOG.md`). An earlier doc claim that every flow's notes drain via `PreTurnBriefingProcessor` was wrong — tool-flow notes used to be silently dropped; the unified list + mapper derivation fixed that.

### `answer` — hand the agent data + a brief, then terminate (agent composes the reply)

```yaml
- answer:
    data: { cases: '$cases' }                   # optional; merged into ToolResponse data (LLM reads it)
    instructions: 'Tell the user each case's status in their dialect.'   # required; a SHORT brief
```

The idiomatic terminal for an **agent-voiced reply built from tool data**. It merges `data`, sets `instructions` (a short brief), emits **no render** (so the agent is NOT suppressed — it composes on the same turn from `data` + `instructions`), then ends `applied`. It is `.strict()` — it **cannot** carry a `memory_note` or a render, so the "prose-in-instructions + redundant memory_note" footgun is structurally impossible.

**Return the values in `data`; keep `instructions` a brief telling the agent what to *do* with them — never hand-format the data into `instructions`.**

### Envelope fields — which to use (cheat-sheet)

The flat tool envelope has four author-settable fields. Reach for the terminal that matches your intent rather than hand-mixing them:

| Field | Reaches the LLM | Use for | Set by |
|---|---|---|---|
| `data` | **this turn** (envelope minus `_render` is JSON the model reads) | structured payload the agent composes from (rows, outcomes) | `answer` · `render` |
| `instructions` | **this turn**, verbatim (agent-protocol §1); **ignored when a render suppresses** (only surfaces via next-turn recall). Last writer wins across nodes | a *short* brief on what to do with `data` | `answer` · `render` |
| `memory_note` | **next turn** — the run's LAST note (unified `memory_notes` list) becomes a `tool_action` system note; pre-turn/media flows drain the whole list as `pre_turn_briefing` instead | a sparse cross-turn breadcrumb — mainly when a `render` suppressed the agent | `render` (`memory_note:` sugar) · `note:` (NOT `answer`) |
| `_render` | **never** (stripped) — presence ⇒ suppression halt (the card *is* the reply) | channel UI (pickers/cards); agent stays silent | `render` · `end: {status, body}` |

**Rule of thumb:** no render → **`answer` (data + short brief)**, no `memory_note`. Render/suppression → the card is the reply; use `memory_note` for next-turn awareness. `end: publish:` publishes **vars** (`$in_turn.*`/cache), *not* envelope `data`.

### `goto` — jump to a named step

```yaml
- goto: validate
```

Target step must exist in `steps:`. Validated at load time.

### `foreach` — iterate an array, running a body per element

```yaml
- foreach:
    in:   '$input.media'        # array expression
    as:   m                     # loop var name (default `item`); also binds `<as>_index`
    body:                       # node-or-seq run once per element
      - do: set_media_metadata
        args: { media_id: '$m.media_id', patch: { analysis: '$analyses[$m_index]' } }
```

Runs `body` once per element of `in`, binding `$<as>` (the element) and `$<as>_index` (0-based)
into a **per-iteration child context** (same as render `for_each`). Side effects in the body
(e.g. `do:` primitive calls) persist; per-iteration `bind`/`assign` are scoped to that iteration
(discarded after — accumulate via the shared envelope, e.g. `note:`, not via `assign`).
A non-`continue` outcome (`goto`/`end`) inside the body **breaks the loop and propagates**.

**Constraint:** the body MUST NOT suspend (`pause`/`collect`). There is no per-iteration resume
point — a suspend would re-run the whole loop from index 0 on resume. Use `foreach` for
run-to-completion work (media-handler flows, batch writes), not for interactive collection.

`in` must resolve to an array (else a load-time/runtime error). Empty array → no-op.

### `track` — emit a customer-journey signal

```yaml
- track: { event: viewed_listing }                 # journey inferred when the id is unique
- track: { event: submitted_lead, metadata: { source: $input.source } }
- track: { journey: support, stage_enter: triage } # `journey:` only needed when an id is ambiguous
- track: { goal_reached: deal_closed, value: 250000, timeline: true }
```

Capture side of the journey-analytics module. Exactly one of `event` / `stage_enter` /
`goal_reached`. The journey is **inferred** from the id against the pack's declared journeys
(`journeys/<id>.journey.yaml`); when the same id appears in 2+ journeys, add `journey: <id>` (an
ambiguous bare id throws at runtime — an authoring bug). An `event` whose spec declares
`advances_to` / `completes` **fans out** into the derived `stage_enter` (with `stage_order` stamped
for the trigger) / `goal_reached` writes — author emits ONE event. `value` / `metadata` resolve via
`resolveValue` (string-is-`$expr`, like `note:`); `value` defaults to the goal's declared
value. `timeline: true` also mirrors a `diagnostic` `journey_event` row into the conversation
timeline. Writes are failure-tolerant (a DB error never fails the turn); `track:` is sugar over the
`track_event` primitive (same writer). Contacts' live state is exposed to flows as
`$journey.<journey_id>.{current_stage, stage_index, completed_goals, goal_value_sum}`.

**Journey declaration + lifecycle.** A journey is declared in
`src/packs/<id>/journeys/<journey_id>.journey.yaml` (stages/goals/events). An optional
`lifecycle:` block sets re-entry semantics (industry-standard names — see
[`24-journey-lifecycle-benchmarks.md`](24-journey-lifecycle-benchmarks.md)), all defaulted to the
lifetime, forward-only, single-run behavior:

```yaml
lifecycle:
  progression:     monotonic     # monotonic (default) | free (stages may move backward)
  reentry:         none          # none (default) | after_exit | anytime
  cooldown_days:   0             # min days before a re-entry opens a new run (instance)
  idle_reset_days: 0             # inactivity TTL → next event starts a fresh run (0 = never)
  terminal:        { goal: deal_closed }   # what marks a run "complete" (required for after_exit)
```

With `reentry` other than `none`, a contact who re-engages after completing the `terminal` opens a
new **run** (`instance_no`); funnel/goals count runs (a re-purchase is a second funnel). Reporting
also supports a **conversion window** (Amplitude-style "convert within N days of entering") via the
admin API `?window=<days>` / the console "Convert within" control.

For the full business + authoring walkthrough (reading the console, declaring a journey, placing
`track:` markers, worked examples), see the **module guide Artifact**:
<https://claude.ai/code/artifact/54836ea5-d38c-40d4-a7c8-b5902bee5f3e>.

---

## §2 Render intents (10)

Source: [`render-intents/intents.ts`](../src/platform/flow-runtime/render-intents/intents.ts).

```
list_picker  buttons_picker  carousel  cta_url  text_card  flow_form
detail_card  entity_detail_card  auto_collection  field_prompt
```

`field_prompt` is a composite (like `auto_collection`): the renderer resolves it at render time
to a `list_picker` when the field carries `options`/`sections` with rows, or a `text_card` (a
free-text ask) when it doesn't. The `collect` engine emits one per field — picker or gap —
so every field is authored identically; caps are applied by the delegate.

**Picker shape vs card shape — they are NOT interchangeable.** Two families author their
actions differently; mixing them fails at *render* time (a `.slice` of undefined), not at parse:

- **Pickers** (`list_picker`, `buttons_picker`) render a *homogeneous* collection: they require
  `items:` (a raw array) **+** `item_template:` (one mapping applied to every row). They have NO
  `buttons:` field. Use when the choices come from data and share one shape.
- **Cards** (`detail_card`, `carousel`) render *pre-composed* content with *heterogeneous*
  actions: `detail_card` takes an explicit `buttons:` array (each `{ title, on_select }`);
  `carousel` takes pre-built `items`. Use for a fixed set of distinct actions (e.g.
  Confirm / Reschedule / Cancel on one card).

So a body + a few fixed action buttons is a **`detail_card`**, not a `buttons_picker` with an
inline `buttons:` list. `buttons_picker` is only for "pick one of N data-driven options that
share a template". (Scheduled `notify` intents are validated against the `UiIntent` schema at
enqueue time by `schedule_notify`, so a wrong-shaped intent fails LOUD on the `failed` port at
flow-run time — `reason: invalid_intent: …` — not silently at delivery.)

### `list_picker` — scrollable list rows (WhatsApp interactive list)

```yaml
render_intent: list_picker
header:        '...'                            # optional, max 60 chars
body:          '...'                            # required
footer:        '...'                            # optional, max 60 chars
cta:           '...'                            # required, max 20 chars (CTA button label)
items:         '$rows'                          # OR use `sections:` for multi-section list
group_by:      area                             # optional — auto-section by a field (see below)
item_template:                                  # optional (identity-spread default)
  title:       '$item.name_ar ?? $item.name'    # full per-row expression — see note
  description: '{$t("…", { n: $item.count })}'  # optional
  on_select:   { invoke: tool, with: { id: '$item.id' } }
sections:                                       # optional multi-section variant
  - title: '...'                                # optional section title
    items: '$rows'
    item_template: { ... }                      # optional; inherits parent's
```

When `item_template` is omitted, the renderer applies an identity-spread default — works if rows already match `{ title, description?, on_select }` (Phase 2 Win 2.4).

**`item_template` is full per-row expression** — it desugars to `for_each` and binds
`$item` to each **raw** row, then evaluates every field through the expression engine
(`$enum_label($item.type,'types')`, `$format_money`, ternaries, `$coalesce`, conditional
`{if,then,else}` fields, per-row `on_select.with`). **Prefer raw `items:` + an inline
`item_template` over a `$map($rows, <mapper-partial>)` step** — it deletes the mapper
partial *and* the intermediate assign. Reserve a partial only for a view-model **shared
by 2+ renders** (or one whose shaped rows also feed the `data:` envelope).

**`group_by: <field>`** buckets `items` into one section per distinct value of `<field>`
(section title = the group key), reusing `$group_by`/`$entries` — replaces the
`$map($entries($group_by($items,"area")), section_wrapper)` idiom and its wrapper partial.
Pure sugar over `sections:` + `for_each`; combine with `item_template`.

### `buttons_picker` — data-driven button choices (one template over N rows)

```yaml
render_intent: buttons_picker
header:        '...'                            # optional
body:          '...'                            # required
footer:        '...'                            # optional
items:         '$rows'                          # required — ≤3 rendered (WA cap)
item_template:                                  # required
  title:     title
  on_select: on_select
```

Picker shape — requires `items:` + `item_template:`, has NO `buttons:` field. For a fixed set
of distinct buttons on a card (Confirm / Cancel / …) use [`detail_card`](#detail_card--header--body--footer--explicit-buttons) instead.

### `carousel` — card scroll

```yaml
render_intent: carousel
body:          '...'
items:         '$cards'                         # ≤10 items rendered
placeholder_url: '$config.placeholders.unit'    # optional; last-resort image URL when channel rejects a card's image
item_template:
  image_url: image_url                          # optional
  body:      body
  buttons:
    - { title: 'Manage', on_select: { invoke: '...' } }
```

`placeholder_url:` is a last-resort image URL channel adapters substitute when a card's resolved `image_url` would be silently dropped (Meta requires `.jpg`/`.jpeg`/`.png` path extensions on `image.link`). Pack authors typically wire it to `$config.placeholders.<entity>`. Web channel ignores this field.

### `cta_url` — link button card

```yaml
render_intent: cta_url
header: '...'                                   # optional
body:   '...'
footer: '...'                                   # optional
label:  'Open'
url:    'https://...'
```

### `text_card` — plain text, no interaction

```yaml
render_intent: text_card
body: '...'
```

### `flow_form` — embed a Meta Flow Form

```yaml
render_intent: flow_form
flow_id:  '...'
header:   '...'
body:     '...'
footer:   '...'                                 # optional
cta_text: '...'
```

### `detail_card` — header + body + footer + explicit buttons

```yaml
render_intent: detail_card
header:  '...'                                  # optional
body:    '...'                                  # can be supplied via `from:`
footer:  '...'                                  # optional
buttons:                                        # can be supplied via `from:`
  - title:     '...'
    on_select: { ... }                          # OR `id: '...'` for legacy prefix-match taps
from:    '$result.data.card'                    # optional; identity-spread shortcut
```

`from:` is a Phase A2 author shortcut. The renderer merges `from`-supplied fields (`{header, body, footer, buttons}`) into the intent; explicit intent fields win. Lets a flow write `{ render_intent: detail_card, from: '$result.data.no_results_card' }` instead of restating every field.

### `entity_detail_card` — switch-case detail_card composite

```yaml
render_intent: entity_detail_card
header: '...'                                   # optional; shared across all branches
body:   '...'                                   # required; shared across all branches
case:   '$viewer_role'                          # discriminator expression
cases:
  owner:
    footer:  '...'
    actions: [ { title: 'Edit', on_select: { invoke: 'edit_listing' } } ]
  buyer:
    footer:  '...'
    actions: [ { title: 'Contact seller', on_select: { invoke: 'relay' } } ]
  default:                                      # magic key — matched when no other case wins
    footer:  '...'
    actions: [ ]
```

The renderer evaluates `case` and dispatches to `cases[String(value)]`, falling back to `cases.default` and then to a body-only card. Replaces the role-conditional `{ if/then/else } → detail_card / detail_card` ladder.

### `auto_collection` — count-shaped composite

```yaml
render_intent: auto_collection
items:  '$items'
empty:  { render_intent: text_card, body: '{$t("...empty")}' }
single: { render_intent: detail_card, body: '$items[0].title', buttons: [...] }
multi:  { render_intent: list_picker, sections: [...], cta: '...' }
```

Dispatches based on `items.length`:
- `0` → `empty`
- `1` → `single`
- `≥2` → `multi`

Each sub-intent is a full valid intent.

**`single` can be an auto-fire directive** instead of a UiIntent:

```yaml
single:
  on_select:                                    # no `render_intent` key — bypasses rendering
    invoke: 'next_step'
    with:   { id: '$items[0].id' }
```

When the runtime detects `on_select` without `render_intent`, it fires the directive inline instead of rendering. Lets cascade-style pickers express "no decision to make, proceed" declaratively.

---

## §3 Expression DSL

Source: [`expr/evaluator.ts`](../src/platform/flow-runtime/expr/evaluator.ts).

### Forms

| Form | Semantics |
|---|---|
| `'$expr'` (bare) | Full expression evaluation. Returns the typed value. First non-ws char is `$`. |
| `'{$expr}'` (interpolated) | Substitution within a string. Result is stringified. Works **uniformly** — at the top level of a render/assign/arg value AND inside an expression-position string literal (a `$lines([...])` element, a ternary branch, a `$map` value): `$lines(['🏠 {$ref} — {$loc}'])` and `$map($rows, { label: '→ {$item.name}' })` both interpolate per the current scope. **Prefer it** over `'…' + $x + '…'` concatenation. No escape for a literal `{$`. |
| `"'…' + $x"` (quote-leading) | Quoted-string-leading **concatenation** expression — first non-ws char is a quote (`'`/`"`), e.g. `"'📅 ' + $coalesce($item.date, '')"`. Evaluated as an expression, NOT literal text. Recognised identically in **render fields, `args:`, and `assign:`** (single-sourced in [`resolveScalarString`](../src/platform/flow-runtime/expr/evaluator.ts)). Prefer `{$…}` interpolation for readability; a literal `$` (currency `$500`) stays literal because it isn't quote-leading. |
| `'t:key'` | Locale shorthand — same as `'{$t("key")}'` |

### Operators

| Operator | Notes |
|---|---|
| `==` / `!=` | Loose equality (matches JS — `$x != null` matches "neither null nor undefined") |
| `===` / `!==` | Strict equality |
| `<` / `<=` / `>` / `>=` | Comparison |
| `+` / `-` / `*` / `/` / `%` | Arithmetic |
| `&&` / `\|\|` | Short-circuit boolean |
| `??` | Null-coalescing (matches JS) |
| `!` | Negation |
| `cond ? a : b` | Ternary (Phase 1 Win 1.2) |

### Member access

- Dot: `$x.y.z`
- Bracket: `$x[0]`, `$x['key']`
- Safe-nav: null/undefined chains return undefined (no error)

### Object / array literals

```yaml
'$t("home.greeting_with_name", { first_name: $summary.first_name })'
'[1, 2, $count + 1]'
```

### Function calls

`$fnName(arg1, arg2)` — only top-level identifiers; no method calls (`$arr.length` / `$arr.join(...)` throw — use `$length($arr)` / `$lines($arr, sep)`). See §6 for builtins.

---

## §4 Output ports — canonical defaults table

Source: [`port-defaults.ts`](../src/platform/flow-runtime/port-defaults.ts).

When `outputs: { <port>: [] }` is declared AND the port has a canonical handler, the runtime synthesizes the standard sequence. Three classes:

| Port name | Class | Mode | Canonical handler |
|---|---|---|---|
| `applied` | A | suppression | `end: applied` |
| `ok` | A | suppression | `end: applied` |
| `aborted` | A | suppression | render `text_card($<bind>.reason)` + `end: aborted` |
| `failed` | A | suppression | render `text_card($<bind>.reason)` + `end: failed` |
| `fail` (alias) | A | suppression | same as `failed` |
| `invalid` | A | complete | `end: { status: failed, instructions: $<bind>.reason }` — **no render**; hands the reason to the AGENT so it fixes its tool input + retries |
| `retry` (alias) | A | complete | same as `invalid` |
| `not_found` | B | suppression | mode-only (pack supplies render — locale/domain-specific) |
| `denied` | B | suppression | mode-only |
| `empty` | B | suppression | mode-only |
| `ready` | B | suppression | mode-only (typically a `goto:` target) |
| `suspended` | B | suppression | mode-only (pause card carries the prompt) |
| `gap` | C | call-site-decides | mode-only (suppression with custom render; complete when render-less) |

Class (A) = platform synthesises a default node sequence when YAML writes `outputs: { <port>: [] }`. Class (B) = platform fixes the reply mode but the YAML supplies the render explicitly (right copy depends on pack locale + domain). Class (C) = the right mode depends on what's attached.

Override A or B by supplying a non-empty body in YAML.

---

## §5 Top-level flow shape

Source: [`schemas/flow.ts`](../src/platform/flow-runtime/schemas/flow.ts).

```yaml
flow_id:        my-flow                         # required; ^[a-z][a-z0-9-]*$
version:        1                               # required; positive int
description:    '...'                           # optional; tool-catalogue description
internal:       false                           # optional; true = platform-only, never exposed to agents
constants:                                      # optional; compile-time vars
  MAX_PAGE_SIZE: 10
config:                                         # optional; runtime-config namespace ($config.<cat>.<key>)
  thresholds:
    list_page_size: 8
  retries:
    embed_max:      3
input:                                          # required (can be `{}`)
  field_name: { type: string, optional: true, describe: '...' }
conversation_context:                           # optional; ambient invocation context
  channel:                { type: string }
  channel_contact_handle: { type: string }
  lang:                   { type: enum, enum: [ar, en, bilingual], default: 'ar' }
require:                                        # optional; runs BEFORE entry routing
  - do: get_user_by_contact_handle
    args: { whatsapp: '$conversation.channel_contact_handle' }
    bind: user
  - if: '$user == null'
    then:
      - render: { render_intent: text_card, body: '{$t("err.unknown_user")}' }
      - end: failed
entry:                                          # required; first matching route wins
  - if: '$input.action != null'
    goto: validate
  - else: true
    goto: render_list
steps:                                          # required; at least one step
  step_id:
    - do: ...
```

**Phase 15 retired the flow-level `cache_ttl_s:` field.** Caching is now declared per-`end:` via `publish: { ..., cached: N }` — a flow with multiple terminal branches can express different cache policies per branch (cache on success, skip on failure).

The legacy top-level `thresholds:`, `notes:`, `tool_behavior:`, and `taps:` fields were also removed — `thresholds` migrated under `config:`, `notes` into `render:` `memory_note:` directly, `tool_behavior:` was retired entirely, and `taps:` was superseded by structured tap-ids (auto-encoded from `on_select:`) + pack-level `<pack>/taps.yaml` (for external id schemes — see [16e-taps-reference.md](16e-taps-reference.md)).

### Field types in `input:` / `conversation_context:`

Source: [`schemas/primitives.ts`](../src/platform/flow-runtime/schemas/primitives.ts).

| `type:` | Notes |
|---|---|
| `string` | |
| `number` | optional `positive: true`, `integer: true`, `min:`, `max:` |
| `boolean` | |
| `enum` | `enum: [a, b, c]` required |
| `array` | `items: { type: ... }` for element shape |
| `object` | `shape: { field: spec }` for nested |

Common modifiers: `optional: true` (DEFAULT), `nullable: true`, `default: <value>`, `describe: '...'`.

> **`optional` defaults to `true`** — tool/flow input fields are overwhelmingly optional. Declare `optional: false` explicitly for the rare required input.

---

## §6 Expression builtins

Source: [`expr/builtins/`](../src/platform/flow-runtime/expr/builtins/) (grouped by domain: object / array / pagination / logic / text / format).

All builtins are pure functions (no I/O). Call as `$fn(...)` in any expression. Mapping arguments (`mapping`, `predicate`, `key`) accept field-name strings or dotted paths — they are NOT expressions evaluated per-item (keeps builtins synchronous and free of evaluator re-entry).

### Tier 0 — Core helpers

| Builtin | Signature | Description |
|---|---|---|
| `$is_empty(v)` | `(unknown) → boolean` | True for null, undefined, '', [], or {} with no keys |
| `$has_any_key(obj, keys)` | `(object, string[]) → boolean` | True when `obj` has a defined value for any key in `keys` |
| `$has_all_keys(obj, keys)` | `(object, string[]) → boolean` | True when `obj` has a defined value for every key in `keys` |
| `$keys(v)` | `(object) → string[]` | Defensive `Object.keys` — empty array for non-objects |
| `$length(v)` | `(unknown) → number` | Array/string length, object key count, else 0 |
| `$merge(a, b)` | `(object, object) → object` | Shallow merge; b's keys override a's. null-safe |
| `$pick(obj, keys)` | `(object, string[]) → object` | Subset of keys from `obj` |
| `$slice(v, start, end?)` | `(arr\|string, number, number?) → arr\|string` | `Array.prototype.slice`; defensive on non-array/string |
| `$take(v, n)` | `(arr\|string, number) → arr\|string` | First N elements / chars |
| `$enum_lookup(value, map, fallback?)` | `(string, object, any?) → any` | Lookup `map[value]`; falls back to `fallback`, then `value` itself |
| `$get(obj, path, default?)` | `(unknown, string, any?) → unknown` | Safe nested lookup by dotted path; returns `default` (or null) when missing — e.g. `$get($constants.mime_extensions, $m.mime_type, '.bin')` |
| `$includes(collection, item)` | `(unknown, unknown) → boolean` | Membership: array element (strict eq) or string substring |
| `$in(item, collection)` | `(unknown, unknown) → boolean` | Reversed-arg membership — reads as `item in collection` |
| `$trim(s)` / `$lower(s)` / `$upper(s)` | `(string) → string` | Trim whitespace / lowercase / uppercase; `''` for non-string |
| `$lines(parts, sep?)` | `(unknown[], string?) → string` | Join non-null parts. Default sep `'\n'`. Empty strings PRESERVED (intentional layout separators); use `.filter(p => p)` to drop them too |
| `$t(key, vars?)` | `(string, object?) → string` | Locale lookup; reads from `PlatformLocaleRegistry` / pack locale |

### Tier 1 — Iteration

| Builtin | Signature | Description |
|---|---|---|
| `$map(arr, mapping)` | `(unknown[], string \| object \| expr \| partial) → unknown[]` | `'name'` → field; `'a.b'` → dotted path; `{ id: 'id' }` → projection; **`{ id: $item.id, label: $enum_label($item.type,'types') }` → computed object per-row** (object VALUES that reference `$item` / call / ternary are evaluated per row, with `$item` bound — the render `item_template`'s power in data position; plain string values stay field projections); **a scalar per-row expression → `$map($missing, $get(LABELS, $item, $item))`** (any non-literal, non-bare-identifier mapper — a call / member / ternary — is evaluated per row with `$item` bound; the scalar twin of the object mapper, replaces single-purpose label/format mapper partials). `$map(arr, partial_name)` runs a render-time partial per row |
| `$filter(arr, predicate)` | `(unknown[], string \| object) → unknown[]` | `'is_active'` → truthy field check; `{ kind: 'apartment' }` → equality match on each pair |
| `$group_by(arr, key)` | `(unknown[], string) → Record<string, unknown[]>` | `'compound'` → group by field value |
| `$sort_by(arr, key, dir?)` | `(unknown[], string, 'asc'\|'desc'?) → unknown[]` | Stable; nulls last regardless of dir |
| `$unique(arr, key?)` | `(unknown[], string?) → unknown[]` | Primitive equality by default; `key` dedupes by `item[key]` |
| `$flat_map(arr, mapping)` | `(unknown[], string \| object) → unknown[]` | Map + flatten one level |
| `$count_by(arr, key)` | `(unknown[], string) → Record<string, number>` | Count occurrences per unique value |
| `$keys_where(obj)` | `(object) → string[]` | Object keys whose values are truthy |
| `$paginate(items, offset, limit)` | `(unknown[], number, number) → { page, total, showing_offset, has_more, next_offset? }` | Window an in-memory array into a page + navigation cursor; `next_offset` omitted on the last page |

### Tier 2 — Composition

| Builtin | Signature | Description |
|---|---|---|
| `$cond(cases)` | `({ when?, default?, then }[]) → unknown` | Multi-way conditional; first match wins. Both `when` and `then` are pre-evaluated |
| `$spread(base, overrides?)` | `(object, { when, value }[]?) → object` | Spread `base` then conditionally merge each override whose `when` is truthy |
| `$shape(obj, mapping)` | `(object, object) → object` | Project + rename fields. Values are field-name strings / dotted paths (NOT expressions) |
| `$array_of(items)` | `(unknown[]) → unknown[]` | Build array with conditional elements: `{ when, value }` objects included only when `when` truthy |

### Tier 3 — Formatting

| Builtin | Signature | Description |
|---|---|---|
| `$format_number(n, locale?)` | `(number, string?) → string` | `Intl.NumberFormat`; default locale `'en-EG'` |
| `$format_money(value, currency, locale?)` | `(number, string, string?) → string` | `Intl.NumberFormat({ style: 'currency' })`; falls back to `${value} ${currency}` on unknown code |
| `$format_relative_date(iso, locale?, base?)` | `(string, string?, string?) → string` | `Intl.RelativeTimeFormat`; picks largest unit with \|Δ\| ≥ 1. `base` for deterministic tests |
| `$truncate(str, max, ellipsis?)` | `(string, number, string?) → string` | Char-based truncation; ellipsis counts against `max`. Default ellipsis `'…'` |
| `$first_n(str, n, ellipsis?)` | `(string, number, string?) → string` | First N WORDS (whitespace-split). Default ellipsis `'…'` |
| `$pluralize(count, forms, locale?)` | `(number, object, string?) → string` | `Intl.PluralRules`; `forms` keyed by CLDR categories (`zero`/`one`/`two`/`few`/`many`/`other`). `{count}` substituted into the chosen template |
| `$coalesce(...args)` | `(...unknown) → unknown` | First non-nullish arg. ⚠️ Returns **`undefined`** (not `null`) when ALL args are nullish — for a null default use the `??` operator (`$x ?? null`), which is the documented null-default idiom |
| `$join_with(arr, sep, last_sep?)` | `(unknown[], string, string?) → string` | Human-readable list join; `last_sep` for "Oxford comma" style — `$join_with(['a','b','c'], ', ', ', and ')` → `'a, b, and c'` |
| `$slug(s)` | `(string) → string` | URL-safe slug; preserves Unicode letters/digits |
| `$unit_ref(uuid)` | `(string) → string` | UUID → `UID-XXXXXXXX` reference |
| `$lead_ref(uuid)` | `(string) → string` | UUID → `LID-XXXXXXXX` lead reference |
| `$summarize_items(arr, template, opts?)` | `(unknown[], string, { max?, separator? }?) → string` | Single-line breadcrumb formatter. Template uses bare `{path}` (not `{$item.path}`). Defaults: `max=5`, `separator=' \| '` |

**Conditional value selection:** use the inline ternary `cond ? a : b` for simple cases; `$cond([...])` for multi-way. `$pick_string` was retired in Phase 1 Win 1.2.

### Runtime-bound functions (pack/locale context, wired by FlowTool per run)

| Function | Signature | Description |
|---|---|---|
| `$t(key, vars?)` | `(string, object?) → string` | Locale lookup, pack-scoped (`<flow_id>.key` / `<pack_id>.key`) |
| `$t_has(key)` | `(string) → boolean` | Missing-key guard without a warning |
| `$enum_label(value, category, fallback?)` | `(string, string, string?) → string` | Auto-namespaced label: `$t("<pack>.<category>.<value>")` — the pack-level enum-label convention |
| `$work_types(entity, route_by?)` | `(string, string?) → string[]` | The pack's declared work sub-type ids for an XRM entity from `worklist.yaml` `work.<entity>.types` (via `WorklistRegistry`); `route_by` filters to types routed by that field (e.g. `$work_types("case", "region")` = the captain case types). `[]` when the pack works no such entity. Single-sources gates like kaiian's region `when:` so the type list is never re-typed in flow YAML |

---

## §7 Tap directives — `on_select`

Used on list rows, buttons_picker buttons, carousel card buttons, detail_card / entity_detail_card buttons, and `auto_collection.single` auto-fire directives.

```yaml
on_select: { invoke: 'tool_name',   with: { id: '$row.id' } }            # direct-tap to a tool
on_select: { resume: '$self',       with: { run_id: '$run_id' }, control: { approved: true } }  # workflow resume
on_select: { reply:  'free text' }                                       # static channel reply
on_select: { agent:  '[TOKEN] human text' }                              # forward synthesized agent-visible token to LLM
```

The four verbs are: `invoke` (direct-tap), `resume` (workflow-resume), `reply` (static text-tap), `agent` (forward to LLM with the synthesized token).

Each accepts two optional binding maps:
- **`with: { ... }`** — DATA field-updates → this turn's `$input`, folded into `$state` / the `collect` accumulator. For picker selections the tap contributes (e.g. `compound_id`).
- **`control: { ... }`** — CONTROL flags (approve/decline/retry/…) → routed to the reserved `$control.*` namespace read in `on_resume`; **never** merged into `$input`/the bag. Encoded under the reserved `_ctl_` binding prefix and split back out by `resumeFlow`. Use it so a decision flag can't pollute persisted data (the publish-listing "approved column" class of bug). Pairs with `resume:`.

Encoded into structured tap-ids (`t:<verb>:<target>:<bindings>`) by [`compileOnSelectsInTree`](../src/platform/flow-runtime/interpreter/render-resolver.ts). Agent tokens auto-derived at compile time.

**`$self`** — sugar for the current flow's `flow_id`. Substituted at compile time.

---

## §8 Templates

Source: [`templates/`](../src/platform/flow-runtime/templates/).

```yaml
- use: approve_apply                            # template name
  with:                                         # template args
    preview: '$preview_body'
    apply:   apply_step_id
    abort:   aborted_step_id
    on_other: validate_step_id                  # optional; user typed during preview
```

Templates expand at load time before schema validation. The walker replaces the `use:` node with the expander's result. Today only `approve_apply` ships from the platform; see [16-platform-stdlib-templates.md](16-platform-stdlib-templates.md) for the catalog (and the pack-override mechanism).

---

## §9 Reserved variables

Available in any `$expr` once the flow is running. Source: [`flow-run-context.ts`](../src/platform/flow-runtime/flow-run-context.ts) for the universal binding, [`interpreter/`](../src/platform/flow-runtime/interpreter/) for the per-step binds.

### Universal (always present)

| Variable | Source |
|---|---|
| `$input.*` | THIS turn's inbound delta — the tool's call args (validated against `input:` schema) on a fresh run, or the resume/tap `with` payload on a resume. **Ephemeral** (not carried across turns); read `$state.*` for a value that must survive a suspend |
| `$state.*` | The run's persistent accumulator — every turn the engine folds `state = { ...state, ...input }` before any node runs. The single home for cross-turn state; a `collect` node accumulates its fields directly into `$state` |
| `$conversation.*` | Ambient invocation context (validated against `conversation_context:` schema). Includes `channel`, `channel_contact_handle`, `name`, `lang`, `conversation_id`, `tenant_id`, `project_id` |
| `$run_id` | Caller-supplied per-invocation id (workflow run id when resuming) |
| `$tool_id` | The flow's `flow_id` |
| `$tenant_id`, `$project_id` | Tenant and project from the inbound boundary |
| `$locale` | Channel locale (`'ar'` / `'en'` / pack-supplied). Defaults to the Contact row's `language` field or `'ar'` |
| `$caps` | Channel hard-limits (`list.maxRows`, `buttons.maxRows`, `carousel.maxCards`, etc.). Read-only — primitives + YAML are channel-blind by convention |

### Conditional (present when supplied)

| Variable | When set |
|---|---|
| `$contact.*` | The platform-resolved `Contact` row (set for tool flows + media-handler flows; absent on pre-turn flows that fire before the contact exists) |
| `$pre_turn.*` | Merged pre-turn pipeline output — published by `pre_turn_flows:` for downstream tool flows + post-turn flows to read |
| `$in_turn.*` | In-loop accumulator — every `end: publish: { ... }` publish from this turn's tool flows so far |
| `$post_turn.*` | Merged post-turn pipeline output — published by earlier post-turn flows in the same pipeline |
| `$inbound.<channel>.<field>` | Per-message channel ephemera from the inbound payload (e.g. `$inbound.whatsapp.context_message_id`) |
| `$config.<cat>.<key>` | Runtime-config namespace declared by the flow's `config:` block |
| `$journey.<journey_id>.*` | Contact's live per-journey state — `current_stage`, `stage_index`, `completed_goals`, `goal_value_sum` (bound for tool flows when the pack declares journeys; emitted by the `track:` node) |

### Per-step / per-mechanic

| Variable | Source |
|---|---|
| `$<primitive_name>` | The most recent `do: <name>` result when `bind:` is omitted (Phase 1 Win 1.3 — `$result` is retired) |
| `$<bind_name>` | Explicit primitive bind via `bind: name` |
| `$control` | CONTROL flags from `on_select.control` (e.g. `$control.approved`, `$control.retry`); read in `on_resume`; NEVER merged into `$input`/`$state`/the collect bag. `{}` on a control-less resume |
| ~~`$resume`~~ | RETIRED — the resume DATA payload is now THIS turn's `$input` directly (folded into `$state` by the turn-merge); `on_resume` reads `$input`/`$state`/`$control` |
| `$item`, `$row` | Iterator vars inside `item_template:` mapping bodies, AND `foreach` (`$<as>` + `$<as>_index`) |

> **`$user` is NOT a platform-reserved variable.** Flows that need a domain `User` record bind it themselves via `do: get_user_by_contact_handle` (or similar pack primitive) with `bind: user`. The `require:` pre-entry sequence is the canonical place for this.

---

## §10 Things that look like fields but are decisions

YAML authors should know the following decisions are made by the runtime, not by them:

- **Suppression / halt** — derived from envelope shape (`_render != null && success === true`); never written in YAML. (`instructions` is null by contract when a render fires, so it isn't a separate halt term.) Phase 15 retired the `_control` wrapper entirely.
- **`workflow_run_id`** — plumbed by FlowTool; never write in YAML.
- **`agent_token`** in `on_select:` — auto-derived by `compileOnSelectFromConcrete`.
- **Tap-id encoding** (`t:invoke:...`, `t:resume:...`) — fully automatic.
- **Item-template field-mapping** — identity-spread default when omitted (`{ title: 'title', description: 'description', on_select: 'on_select' }`).
