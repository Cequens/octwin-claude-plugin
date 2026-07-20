# Flow reference (`flows/tools/<flow>.flow.yaml`)

A flow is one agent-callable capability. Three parts:

```yaml
flow_id:     main
version:     1
description: |-
  When to call this tool (the agent reads this). One or two sentences.

input:                       # the tool's SCHEMA — drives the LLM tool catalogue + validation
  message: { type: string, optional: true, describe: 'Optional note from the customer.' }

entry:                       # routing: pick the first branch whose condition holds
  - else: true
    goto: greet

steps:                       # each step is a list of nodes; each node has exactly ONE op-key
  greet:
    - assign:
        body: '$coalesce($input.message, $t("main.default"))'
    - render:
        render_intent: text_card
        body: '{$body}'
      memory_note: '{$t("main.note")}'
    - end: applied
```

## Nodes (one op-key each)

`do` (call a built-in) · `render` (compose UI) · `assign` (compute a value) · `if` / `branch`
(conditionals) · `collect` (multi-field intake) · `foreach` · `goto` · `end` · `pause` · `answer`
(hand back to the agent with instructions) · `note` · `track`.

## `do:` calls a platform built-in

A pure-YAML pack never writes code — every `do:` names a **built-in** the platform ships. The COMPLETE,
current list (with exact inputs + output ports) is in the platform KB — see [capabilities.md](capabilities.md)
(`primitives.json` / `primitives-guide.md`); verify a name there before using it. Common ones:

- **Records** (with `xrm.yaml`): `record_save`, `record_get`, `record_list`, `record_search`,
  `record_stage`, `record_note`, `task_open` / `task_complete` / `task_list`.
- **Tickets / casework** (with `worklist.yaml`): `case_open`, `case_get`, `case_list`, `case_reply`, `case_transition`.
- **Messaging / misc:** `notify` (send a proactive message), plus catalog/cart/order and scheduling
  built-ins when those modules are declared.

```yaml
- do: record_list
  args: { entity: model, all: true, limit: 20, sort_by: { field: price, dir: asc } }
  bind: page                 # with `outputs:`, `page` = the step's data payload
  outputs:
    ok:     [{ goto: show }]
    empty:  [{ goto: none }]
    failed: [{ end: failed }]
```

**Ports.** A branching built-in returns through a named **port** (`ok` / `empty` / `failed`, etc.);
the `outputs:` map routes each. Two rules:
- **A port body must END in a terminal node** — `goto`, `end`, or `pause`. `render` / `answer` /
  `assign` are **not** terminal; put them in a named step and `goto` it (e.g. `empty: [{ goto: none }]`
  → a `none:` step that renders).
- **No-result is a port, not a null check** — route `empty`, don't `if`-check afterward.

## Expressions

Values in `args:` / `assign:` / `render:` are expressions. A string containing `{$...}` is a template
(`'{$body}'`, `'Found {$page.total} results'`); otherwise use a bare expression (`'$page.rows'`).

- Built-in functions: `$t("<flow>.<key>", { vars })`, `$coalesce(a, b)`, `$trim(s)`, `$length(x)`,
  `$lines(list, sep)`, `$compact(list)` (drop empties), `$format_number(n)`, `$enum_label(field, v)`.
- **No method calls.** `$arr.length` / `$arr.join()` are invalid — use `$length($arr)` /
  `$lines($arr, ", ")`. Only top-level function calls and `.field` access are allowed.
- **Money fields** are stored as `{ value, offset }` (amount = `value/offset`) — render with
  `$format_money_field($item.fields.price, "EGP")`, NOT `$format_number` (which blanks the bag).

## Context available in every flow

Read ambient data through these namespaces (the full contract is in the platform KB — `runtime.md`):

- `$input.*` — the tool's input fields (your flow's `input:` schema) + a resumed tap's `with:` payload.
- `$state.*` — the `collect:` accumulator and anything you `assign`.
- `$contact.*` — the resolved contact: `$contact.id`, `$contact.channel`, `$contact.channel_contact_handle`,
  `$contact.display_name`, `$contact.language`, and namespaced `$contact.metadata.<ns>.<field>` (e.g. a
  pack-user id). Unbound (absent) before the contact exists — e.g. in a pre-turn flow.
- `$conversation.*` — the ambient fields declared in the manifest's `default_conversation_context`
  (e.g. `$conversation.lang`).
- `$config.*` — pack/flow `config:` values.
- your `bind:` name — a `do:` step's data payload (e.g. `$page.rows`).

## Render intents (the UI)

Set `render_intent:` on a `render:` node. The renderer applies per-channel caps — **you never clamp,
page, or read a channel limit in YAML.** Valid intents include `text_card`, `list_picker`,
`buttons_picker`, `carousel`, `detail_card`, `cta_url`, `flow_form`.

- **Picker vs card.** Pickers (`list_picker`, `buttons_picker`) render a homogeneous collection and
  require `items:` + `item_template:`. Cards (`detail_card`, `carousel`) render pre-composed content;
  `detail_card` takes an explicit `buttons:` array.
- **Text list vs image carousel** — pick by whether the rows carry a photo. No image → `list_picker`.
  Each row has a photo → `carousel` with an `item_template` mapping the image field into `image_url:`.

Full contrast + the `record_list` page shape (`{ rows, total, … }`, `all:`, `active_only`, `sort_by`):
[data-and-render.md](data-and-render.md).

## Locale (`flows/tools/<flow>.locale.<lang>.yaml`)

Every user-visible string lives here, referenced as `$t("<flow>.<key>")`. Keys are **namespaced**
(`$t("main.greeting")`) — a bare `$t("greeting")` returns the literal key and fails validation, as does
a `$t` key with no matching locale entry.

```yaml
# flows/tools/main.locale.ar.yaml
main:
  default: 'Hello! How can I help you today?'
  note:    'Greeted the customer.'
```

## Multi-field intake — `collect:`

To gather several fields (name, phone, date…), use a `collect:` node, not a hand-rolled loop: declare
the fields with optional pickers, validation, and media-upload fields — the platform runs the back-and-forth.

## Render a picker and wait for a tap (the portable way — no `collect` engine)

For a **one-off** choice, render a picker and `pause:` for the tap. This is the portable alternative to
the `collect: … options:` sugar (which also works) — a `render:` node whose rows carry `on_select`, then a
`pause: { on_resume }` where the flow continues. The tap's `with:` payload is merged into `$input` on resume:

```yaml
  pick:
    - do: slot_list
      args: { resource_record_id: '$input.model_id' }
      bind: slots
      outputs: { ok: [{ goto: offer }], empty: [{ goto: none }], failed: [{ end: failed }] }
  offer:
    - render:
        render_intent: list_picker
        header: '{$t("book.pick_slot")}'
        items:  '$slots'                       # slot_list `ok` is a bare array
        item_template:
          title: '{$item.local_date} {$item.local_time}'
          on_select: { resume: '$self', with: { slot_start: '$item.slot_start' } }
      memory_note: 'Offered slots.'
    - pause: { on_resume: [{ goto: confirm }] }  # resumes HERE; $input.slot_start is now set
  confirm:
    - do: booking_reserve
      args: { resource_record_id: '$input.model_id', slot_start: '$input.slot_start' }
      outputs: { ok: [{ goto: done }], full: [{ goto: offer }], invalid_slot: [{ goto: offer }], failed: [{ end: failed }] }
```

`on_select` verbs: **`resume: '$self'`** (continue THIS flow after the `pause`), **`invoke: <flow>`** (route to
another flow — no `pause` needed), `reply` / `agent` (hand back to the agent). Full contract: platform KB
`dsl.md` §render / §pause.
