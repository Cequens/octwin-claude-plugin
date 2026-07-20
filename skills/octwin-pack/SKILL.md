---
name: octwin-pack
description: Author a pure-YAML Octwin pack — a WhatsApp/web conversational bot declared entirely in YAML — and deploy it to your tenant with octwin-cli. Use when the user wants to build, scaffold, or extend an Octwin pack (its agent, conversation flows, and data) for their own business, with no platform checkout and no TypeScript.
---

# Authoring an Octwin pack (pure-YAML)

An **Octwin pack** is a self-contained conversational bot for WhatsApp + web — its agent,
its conversation **flows**, its prompts, and its data model — declared **entirely in YAML**.
You build it in your own repo and deploy it to a running Octwin platform with the
[`octwin`](https://www.npmjs.com/package/octwin-cli) CLI; the platform runs it on your tenant.

This skill takes you from nothing to a deployed, chattable pack. Two rules frame everything:

- **Pure declarative data only.** A pack is `.yaml` / `.md` / `.sql` / `.json` — **no `.ts`/`.js`,
  no custom code, no primitives**. Flows are composed from the platform's **built-in** steps. This
  is what lets the platform safely run your pack; the server rejects any code on deploy.
- **Scaffold, then customize.** Never hand-write pack boilerplate from memory — start from
  `octwin init` and edit. The scaffold is a real, valid, chattable hello-bot.

## Step 0 — Scaffold + read what you got

```bash
npx octwin-cli@latest init ./my-pack --id my-pack --description "What the bot does"
cd ./my-pack
```

This writes a minimal, valid pack. **Read every file** before editing — the layout IS the contract:

```
manifest.yaml                       # the whole pack is declared here (id, agent, flows, channels)
flows/tools/main.flow.yaml          # one agent-callable capability = one flow (the DSL)
flows/tools/main.locale.ar.yaml     # every user-visible string, keyed (referenced via $t)
prompts/identity.md                 # the agent's system prompt (who it is, when to call tools)
pack.json                           # deploy target (platform_url, tenant, project) — not part of the pack
```

## Step 0.5 — The capability reference is the source of truth (not this skill's prose)

The exhaustive list of what the platform supports — every built-in `do:` primitive, every expression
builtin (`$…`), every `render_intent`, the full flow-DSL schema, **the declaration schemas for what you
write** (`manifest.yaml`, `xrm.yaml`, `worklist.yaml`/casework, `scheduling.yaml`, …), **the platform's
built-in system entities** (`product`/`cart`/`order`/`case`/`booking`), and the module authoring docs —
is **not** in this skill. It lives in the **platform KB**, which ships with this skill and can be refreshed live:

- **Bundled snapshot** (always available, offline): [`references/platform-kb/`](references/platform-kb/) —
  markdown docs (`primitives-guide.md`, `dsl.md`, `runtime.md`, …) + `*.json` catalogs.
- **Live pull** (matches YOUR platform's exact version — do this once `pack.json` + `octwin login` are set):
  ```bash
  octwin platform-kb pull        # → .octwin/platform-kb/  (same layout as the bundled snapshot)
  ```

**Consult it whenever you need to know whether a step / function / render-intent exists or its exact
fields** — read the markdown first, then the `*.json` catalogs for precise schemas. **Never invent a
built-in from memory or copy a list out of this skill's prose** — the examples below are illustrative,
not exhaustive; the KB is authoritative.

## Step 1 — Author the domain

Decisions to make, each with its contract (details in the references):

### Manifest (`manifest.yaml`)
Declares the pack: `id`, `version` (a **string**, e.g. `'1.0.0'`), `description`, `supported_channels`,
`required_adapters` (a bot with no data of its own needs only `[messaging]`), `default_settings.locale`,
`flows`, and one or more `agents`. Optional record modules are declared by a **sibling file** next to
the manifest — `xrm.yaml` (records with stage pipelines) / `worklist.yaml` (support tickets / casework). Full shape:
[references/manifest.md](references/manifest.md).

### Flows (`flows/tools/<flow>.flow.yaml` + `<flow>.locale.<lang>.yaml`)
Each agent-callable capability is a **flow** — `input:` (the tool's schema) → `entry:` (routing) →
`steps:` (nodes). A step's `do:` calls a **platform built-in** (`record_save`, `record_list`,
`db_rpc`, `notify`, …); a `render:` node composes the on-screen UI (`text_card`, `list_picker`,
`buttons_picker`, `carousel`, …). *(Those are examples — the COMPLETE built-in + render-intent lists
are in the platform KB, Step 0.5. Verify names/fields there before using one.)* Hard rules that apply
to pure-YAML packs:
- **All user-visible copy lives in the locale file**, referenced as `$t("<flow>.<key>")`. Never inline.
- **The renderer owns channel limits** — never `.slice()` to a UI cap or hardcode "10 rows". Return all
  the data + one render intent; the platform clamps/pages per channel. Litmus: *would the value change
  if WhatsApp were swapped for web?* Yes → it's the renderer's job.
- **The tool's return envelope drives the next turn**, not the prompt — a `render`/`memory_note` on the
  returned envelope tells the agent what happened.
The node set, the expression grammar (`$t`, `$coalesce`, `$length`, …), ports, and render intents:
[references/flows.md](references/flows.md).

### Where the data lives — use a first-class module, not a database
Most packs need **no database**. Declare an `xrm.yaml` (records with pipelines — a lead, a booking,
a catalog of models) or `worklist.yaml` (support tickets / casework) and the platform gives you storage + the
`record_*` / `case_*` built-ins for free. This is also how you **seed demo data** (with optional
AI-generated images). See [references/data-and-render.md](references/data-and-render.md).

### Agent (`prompts/identity.md` + `manifest.agents[]`)
The agent expresses **domain intent** and calls flow-tools; it stays channel-agnostic. The prompt says
*when* to call a tool; the tool's returned envelope controls *what happens next*. Keep it brief.

## Step 2 — Validate locally

```bash
octwin validate            # structure + pure-YAML rules, offline
```

Fix anything it flags. The platform re-validates the **full** manifest + flow schema on deploy, so a
local pass is necessary but the deploy is the final word.

## Step 3 — Deploy + test on your tenant

```bash
# one-time: point pack.json at your platform + tenant, then log in with a deploy token
#   (Octwin console → your workspace → API tokens → Generate)
octwin login --url https://your-octwin.example.com --token oct_…

octwin deploy --seed       # --seed also loads any demo data your xrm.yaml declares
octwin status              # "✓ live and current" once it's warm
# → chat with it on your tenant (web widget / console test page)
```

Edit and `octwin deploy` again — a redeploy **hot-loads with no restart**; re-run `octwin status`
to confirm the live version caught up.

## Guardrails

- **No code.** `.ts`/`.js`, `routes/`, DB clients, and `*.primitives.*` are rejected on deploy. If a
  flow seems to need custom logic, express it with built-in steps + the expression grammar, or model
  the data with `xrm.yaml`. (Custom primitives are a platform-internal capability, not an external one.)
- **Don't invent structure** — scaffold with `octwin init` and edit. When unsure of a field, check the
  references here rather than guessing; the manifest is strict and rejects unknown keys.
- **Locale everything.** A bare or missing `$t` key fails validation.
- **Renderer owns caps/paging/splitting.** Never read a channel limit in your YAML.
- **Version is a string.** `version: '1.0.0'` — a bare `1` is a number and fails.
