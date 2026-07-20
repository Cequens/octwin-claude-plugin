# Manifest reference (`manifest.yaml`)

The manifest declares the entire pack. It is **strict** — any key the platform doesn't recognize is
rejected on deploy. Keep `octwin validate` green and let the deploy be the final check.

## Required keys

| Key | Type | Notes |
|---|---|---|
| `id` | string | lowercase, hyphenated; matches the pack directory name |
| `version` | **string** | e.g. `'1.0.0'` — a bare `1` is a number and fails |
| `description` | string | one line; shown in the console |

## Common optional keys

| Key | Shape | Purpose |
|---|---|---|
| `supported_channels` | `string[]` | e.g. `[whatsapp, web]` |
| `required_adapters` | `string[]` | platform capabilities the pack needs. A pack with no data of its own needs only `[messaging]`. |
| `default_settings` | `{ locale }` | default language for `$t` lookups + platform copy (e.g. `ar` or `en`) |
| `flows` | `string[]` | flow ids → `flows/tools/<id>.flow.yaml` (+ its locale file) |
| `agents` | `agent[]` | see below (usually one) |
| `config` | `{ category: { key: value } }` | pack-level values merged into every flow's `config:` |
| `default_conversation_context` | record | fields every flow inherits (e.g. `lang`) |
| `pre_turn_flows` / `post_turn_flows` | `string[]` | flows run once per inbound / after a completed turn |
| `inbound_preprocessing` | `{ image?, voicenote?, document? }` | turn on built-in media analysis (vision / speech-to-text) per inbound media kind — no flow needed |

## `agents[]` (one bot)

```yaml
agents:
  - id:                assistant                # required
    display_name:      'My Assistant'           # required
    default_model:     'openrouter/anthropic/claude-sonnet-4-5'   # required
    tools:             [main]                    # flow ids this agent may call
    include_platform_protocol: true              # keep true — appends the universal agent protocol
    instructions:                                # parts joined with a blank line
      - file: prompts/identity.md                # pack-root-relative markdown
    working_memory:                              # optional — a running scratchpad
      populate: |-
        # Customer
        - Name: {{channel.display_name}}
```

The agent expresses **domain intent** and calls its flow-tools; the platform decides the channel UI.
Keep the prompt brief — the tool's returned envelope drives the turn-by-turn behavior.

## Record modules — declare a sibling file, get storage for free

Most packs need **no database**. Drop a YAML file next to `manifest.yaml` and the platform provides
the storage, a console surface, and built-in flow steps — no migrations, no DB.

| File | For | Built-in flow steps it unlocks |
|---|---|---|
| `xrm.yaml` | **records with a lifecycle** — a lead, a booking, an application, or a reference catalog (products/models) | `record_save`, `record_get`, `record_list`, `record_search`, `record_stage`, `record_note`, `task_open`, `task_complete`, `task_list` |
| `worklist.yaml` | **support tickets / casework** — queues + work-item types (routing, SLA, dispositions); the built-in `case` type gets a casework preset (there is no `cases.yaml`) | `case_open`, `case_get`, `case_list`, `case_reply`, `case_transition` |

`xrm.yaml` also carries a **`demo:`** block that seeds reference/demo records at deploy time (with
`octwin deploy --seed`), and a field can be `'generate:<image prompt>'` to attach an AI-generated
photo (needs the `media:generate` scope on your deploy token). Full shape + the `record_list` /
carousel patterns: [data-and-render.md](data-and-render.md).

## Not allowed in an external pack

These mean executable code or a private capability and are **rejected on deploy** (pure-YAML only):
`.ts`/`.js` files, `routes/`, `db/client.*`, and `*.primitives.*`. Model logic with built-in flow
steps + the expression grammar, and model data with `xrm.yaml` / `worklist.yaml`. The authoritative
schemas for every declaration file (manifest, xrm, worklist, scheduling, …) are the `declarations`
catalog in the platform KB (see Step 0.5), and the platform's built-in entities are in `system-entities`.
