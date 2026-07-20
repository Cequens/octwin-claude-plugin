# Data & rendering reference

Your pack needs **no database**. Declare an `xrm.yaml` next to `manifest.yaml` and the platform stores
your records and gives you the `record_*` flow steps.

## `xrm.yaml` — records with (or without) a pipeline

```yaml
entities:
  # A PIPELINE entity: moves through stages. Use for a lead, a booking, an application.
  - name: lead
    fields:
      - { name: name,  type: text, required: true }
      - { name: phone, type: text }
    pipeline:
      stages: [new, contacted, won, lost]

  # A REFERENCE entity: no pipeline, so it always lists. Use for a catalog/lineup.
  - name: model
    fields:
      - { name: name,  type: text, required: true }
      - { name: code,  type: text }
      - { name: price, type: money }
      - { name: photo, type: text }        # holds an image reference (see demo: below)

# Optional: seed reference/demo records at deploy time (with `octwin deploy --seed`).
demo:
  - entity: model
    fields:
      name:  'Model X'
      code:  x
      price: 2100000
      # 'generate:<prompt>' → an AI-generated photo, IF the deploy token has the media:generate scope.
      # Without that scope the record still seeds, just without the image.
      photo: 'generate:A studio product photo of a silver electric SUV, white background, high detail.'
```

## `record_list` — listing records for a flow

`record_list` returns a **page**: `{ rows, total, offset, limit, has_more, next_offset }`. Bind it with
an `outputs:` switch and read `$page.rows` (render-ready rows: `{ id, title, subtitle, stage, fields }`)
and `$page.total`. **Two defaults bite if you forget them:**

- **It is scoped to the current conversation's contact by default.** For a shared catalog/lineup
  (models, branches, a menu) pass **`all: true`** — otherwise it returns empty or errors with
  *"no contact context"*.
- **`active_only` defaults to `true`** — records in a *terminal* pipeline stage are hidden. For
  always-visible reference records, give the entity **no pipeline** (as `model` above).

Order with `sort_by: { field: <declared field>, dir: asc|desc }`.

## Browse flow — text list vs image carousel

Render `$page.rows` through a picker or a carousel. Pick by whether the rows carry a **photo**:

```yaml
steps:
  load:
    - do: record_list
      args: { entity: model, all: true, limit: 20, sort_by: { field: price, dir: asc } }
      bind: page
      outputs:
        ok:     [{ goto: show }]
        empty:  [{ goto: none }]
        failed: [{ end: failed }]
  show:
    - render:
        render_intent: carousel                 # photos → carousel; no photos → list_picker
        body:  '{$t("browse.intro")}'
        items: '$page.rows'
        item_template:
          image_url: '$coalesce($item.fields.photo, "")'   # a photo ref resolves to an image URL
          body:      '$lines($compact([$item.title, ("From " + $format_number($item.fields.price))]), "\n")'
          buttons:
            - { title: '{$t("browse.book")}', on_select: { invoke: book, with: { model_code: '$item.fields.code' } } }
      memory_note: 'Showed the lineup ({$page.total} items).'
    - end: applied
  none:
    - answer:
        instructions: 'The lineup is empty. Apologize briefly and offer to take the customer's details.'
```

- `image_url:` takes the record's image field directly — the renderer turns a stored image reference
  into a URL, and a row with no photo just renders as text.
- A row `buttons[].on_select: { invoke: <flow>, with: { … } }` routes a tap straight to another flow.
- **Never** `.slice()` the rows to a channel cap or read "how many rows fit" — the renderer clamps and
  pages per channel. Return everything + one render intent.

## Support tickets / casework — `worklist.yaml`

For a "problem to resolve" (not a pipeline you own), declare **`worklist.yaml`** with `queues` + `work`
item types (routing, SLA, dispositions) — the platform ships a `case` system entity and a **casework
preset**, so casework is a preset over the worklist engine (there is no `cases.yaml`). Use `case_open` /
`case_reply` / `case_transition` / `case_list` in flows. Tickets are worked by humans in the Octwin
console and mediated to the customer by the agent. Exact schema: the `worklist` entry in the platform
KB `declarations` catalog + the casework guide (Step 0.5).
