# Platform Stdlib Templates

Three YAML templates ship with the platform and auto-register at boot. Any pack flow can `use: <name>` without per-pack discovery wiring.

| Template | Purpose |
|---|---|
| [`approve_apply`](#approve_apply)     | Suspend with approve/decline + 3-way resume routing |
| [`gap_collect`](#gap_collect)         | Suspend with a prompt; resume into a target step |
| [`cascade_picker`](#cascade_picker)   | Fetch list → auto_collection (empty / single-autofire / multi) |

All templates live under [`src/platform/flow-runtime/templates/`](../src/platform/flow-runtime/templates/). The catalog is emitted to `dist/octwin-platform-kb/templates.json` at build time.

> **Retired stdlib:**
>
> - `entity_card` moved to [`src/platform/partials/entity_card.partial.yaml`](../src/platform/partials/entity_card.partial.yaml) (it produced a value, not flow structure — see [Templates vs Partials](#templates-vs-partials) below; partial catalog at [16a-platform-stdlib-partials.md](16a-platform-stdlib-partials.md)).
> - `disambiguate` / `resolve_ref` were removed entirely — no pack adopted them.
> - `with_lookup` / `with_validation` / `with_idempotency` were retired in the templates audit. Each wrapped a single `do:` + `outputs:` node and only renamed `outputs.<port>` → `on_<port>` — no abstraction value beyond the grammar-level switch table. Pack callsites inlined to direct `do:` + `outputs:` blocks; canonical port-default handlers (`outputs.failed: []` etc., see [port-defaults.ts](../src/platform/flow-runtime/port-defaults.ts)) cover the same defaults declaratively.

---

## Templates vs Partials

The platform has two reusability mechanisms. They look similar at a glance — both ship YAML stdlib units, both register at boot, both pack-overridable. They serve **structurally different jobs** and aren't substitutable.

| | Templates (`use: <name>`) | Partials (`$partial(name, …)`) |
|---|---|---|
| **Where it lives** | `src/platform/flow-runtime/templates/*.template.yaml` | `src/platform/partials/*.partial.yaml` |
| **When it runs** | Flow-load time (before schema validation) | Render time (during expression evaluation) |
| **What it emits** | A subtree of FLOW NODES — `suspend`, `branch`, `do`, `goto`, `end`, `render`, `assign` | A VALUE — typically a string, render-intent object, or array (rows / sections) |
| **What it can see** | Only `$arg.X` (the `with:` block, before expressions evaluate) | Full runtime context — `$contact`, `$agent`, `$<bind>`, `$item`, params, builtins |
| **Invocation** | `- use: template_name`\n` with: {…}` | `'$partial(name, {…})'` in any expression position |
| **Discovery** | [`discoverPlatformTemplates`](../src/platform/flow-runtime/templates/yaml-expander.ts) + [`discoverYamlTemplates(packRoot)`](../src/platform/flow-runtime/templates/yaml-expander.ts) at pack `definePack()` | [`discoverPlatformPartials`](../src/platform/flow-runtime/partials/loader.ts) + [`discoverPackPartials(packRoot)`](../src/platform/flow-runtime/partials/loader.ts) at pack `definePack()` |

### Rule of thumb

> **Need flow control (suspend, branch, do, goto)? Use a template. Need to compute a value (string, render intent, row pattern, formatted label)? Use a partial.**

If you're tempted to wrap a `branch:` whose only job is to pick between two values that then feed a single `render:`, that's value-shaped — write a partial. That's exactly the move that retired the old `entity_card` template.

### What about `use:` inside a partial body, or `$partial(…)` inside a template arg?

Both work, and both are legitimate. A pack-side template that wraps `approve_apply` can pass `'$partial(my_preview, …)'` as the `preview:` arg — the string evaluates at render time when the suspend resolves its preview field. A partial body can include a `t:flow.key` shorthand or call any builtin. The mechanisms compose because templates run earlier than partials.

### Pack-side reusables: should you author one?

The discovery infrastructure works symmetrically for partials and templates — `<pack-root>/partials/*.partial.yaml` and `<pack-root>/templates/*.template.yaml`. **Don't author either speculatively** — wait until you see actual duplication (the same 5–10 lines repeated across 3+ flows with only arg values differing). Premature abstraction hides the flow's story behind an `use:` / `$partial(…)` chase. The realstate pack has zero pack-side templates today; that's a healthy sign — the universal stdlib covers it.

When you do author one, follow the same rule of thumb: structure (control flow) → template, value (string / intent / row) → partial.

---

## How template discovery works

[`discoverPlatformTemplates()`](../src/platform/flow-runtime/templates/yaml-expander.ts) fires at module load and scans `src/platform/flow-runtime/templates/*.template.yaml`. Each file is parsed and registered with the process-level `templateRegistry`. Packs may override by name — `templateRegistry.register()` replaces — so a pack can ship its own e.g. `gap_collect.template.yaml` if the universal one doesn't fit.

---

## `approve_apply`

Suspend with an approve/decline render and resume into one of three step ids based on the user's choice. Replaces the suspend → on_resume → branch-on-`$control.approved` boilerplate.

### Params

| Name | Type | Required | Description |
|---|---|---|---|
| `preview`              | string | ✓ | Preview body shown to the user |
| `apply`                | string | ✓ | Step id to goto when approved |
| `abort`                | string |   | Step id to goto when declined (XOR `abort_render`) |
| `abort_render`         | object |   | Inline render to show on decline + end aborted (XOR `abort`) |
| `abort_path`           | string |   | Path label for the inline abort render |
| `on_other`             | string |   | Step id when resume is neither approve nor decline |
| `header`, `footer`, `confirm_title`, `cancel_title`, `memory_note` | various | | Pass-through render metadata (the `memory_note` rides the approve/decline render) |

### Example

```yaml
- use: approve_apply
  with:
    preview: '$preview'
    apply:   apply_changes
    abort:   aborted
    on_other: collect
    path:    confirm
```

---

## `gap_collect`

Render a gap prompt (text_card OR render-less via `message_ar`), suspend, and on resume goto a target step. The resume payload is this turn's `$input`, folded into `$state` by the turn-merge before the target step re-runs; read `$state.<field>` for values that must survive the suspend.

### Params

| Name | Type | Required | Description |
|---|---|---|---|
| `prompt`       | string  |   | Body shown in the gap card. EITHER `prompt` OR `message_ar` must be supplied. |
| `message_ar`   | string  |   | Agent-narrated prompt for render-less gaps |
| `instructions` | string  |   | Agent instructions for the next typed turn |
| `schema`       | object  | ✓ | Resume payload zod-lite spec — e.g. `{ field: 'type?' }` |
| `resume_to`    | string  | ✓ | Step id to goto on resume |
| `path`, `header`, `footer`, `memory_note`, `data`, `narrate`, `raw_resume` | various | | Optional pass-through |

### Example

```yaml
- use: gap_collect
  with:
    prompt: '$validation.prompt'
    schema: { offer_amount_egp: 'number?' }
    resume_to: validate
```

---

## `with_*` templates — retired

The `with_lookup` / `with_validation` / `with_idempotency` templates were retired in the templates audit. Each wrapped a single `do:` + `outputs:` node and only renamed `outputs.<port>` → `on_<port>` — no abstraction value beyond the grammar-level switch table. Inline `do:` + `outputs:` blocks with canonical port-default handlers cover the same patterns directly:

```yaml
# Inlined replacement for `use: with_lookup`:
- do:   get_thing_by_id
  args: { id: '$input.id' }
  bind: thing
  outputs:
    not_found:
      - render: { render_intent: text_card, body: 'not found' }
      - end: failed
    found:
      - render: { render_intent: detail_card, body: '$thing.summary' }
      - end: applied

# Inlined replacement for `use: with_validation` (note `fail: []` triggers
# the canonical port-default handler — renders `$validation.reason` + end failed):
- do: validate_X
  args: { ... }
  bind: validation
  outputs:
    fail:  []
    gap:   [ use: gap_collect, with: { resume_to: validate } ]
    ready: [ goto: approve ]

# Inlined replacement for `use: with_idempotency`:
- do: do_publish
  args: { state: '$state' }
  bind: result
  outputs:
    existing: [ render: { ... }, end: applied ]
    new:      [ render: { ... }, end: applied ]
    failed:   [ suspend: { ... } ]
```

The canonical port-default handlers ([port-defaults.ts](../src/platform/flow-runtime/port-defaults.ts)) cover the "render `$<bind>.reason` + end failed" pattern automatically when you supply an empty array `[]` for `fail` / `failed` / `aborted` ports.

---

## `disambiguate` / `resolve_ref` — retired

Both templates were retired as unused stdlib. No pack adopted them across the lifetime of the platform; shipping speculative stdlib violates the same "wait for the third occurrence before factoring" rule pack authors are asked to follow. Removed alongside their tests + catalog entries.

If you find yourself reaching for either pattern in a future pack, the inline equivalents are short:

- **Count-shaped picker** — use [`auto_collection`](16a-platform-stdlib-partials.md#auto_collection-render-intent) directly in your render with explicit `empty / single / multi` sub-intents (it's a few lines and you'll want pack-specific copy anyway).
- **Resolver port-table dispatch** — write the `do:` with an `outputs: { one, many, none }` block directly in the flow YAML.

If the pattern recurs across 3+ flows in your pack, factor it into a *pack-side* template — the platform stdlib is only for cross-pack universal patterns.

---

## `entity_card` (retired — now a partial)

The `entity_card` template was retired in favour of the [`entity_card`](16a-platform-stdlib-partials.md#entity_card) partial. The legacy template emitted a `branch:` whose sole job was to pick between two `assign:` blocks that fed one `render:` — value-shaped work in template clothing. As a partial it's a single ternary that returns a `detail_card` UiIntent. See [Templates vs Partials](#templates-vs-partials) for the framing.

Migration: replace `- use: entity_card\n  with: {…}` with `- render: '$partial(entity_card, {…})'` (same arg names, same semantics). One existing call site (`lead-action.flow.yaml`) was migrated when the template retired.

---

## Adding a new platform stdlib template

1. Add `src/platform/flow-runtime/templates/<name>.template.yaml`.
2. The file MUST declare `template_id: <name>` matching the filename stem.
3. Run `npm run platform-kb:templates` to refresh the catalog.
4. Add a test in [`yaml-expander.test.ts`](../src/platform/flow-runtime/templates/yaml-expander.test.ts) — fetch via `templateRegistry.get('<name>')` and assert the expanded body shape.

Templates auto-register at module load — no per-pack wiring needed.

## Catalog

A machine-readable catalog of every platform stdlib template (name + summary + param schema) is emitted to `dist/octwin-platform-kb/templates.json` on `npm run build`. AI authoring tools consume this catalog as LLM context.
