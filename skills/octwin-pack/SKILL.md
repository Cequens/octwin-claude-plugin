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

## Step 0 — Install the CLI, scaffold, read what you got

Install the CLI **once** — a fast, stable `octwin` command that also nudges you when a newer version ships:

```bash
npm i -g octwin-cli        # install once; then use `octwin <cmd>` everywhere below
octwin --version           # confirm it's on the current release
# No-install fallback (CI, or can't install globally): npx octwin-cli@latest <cmd>
```

Scaffold a pack and read it:

```bash
octwin init ./my-pack --id my-pack --description "What the bot does"
cd ./my-pack
```

This writes a minimal, valid, chattable pack that already models the **home-hub** shape (Step 1). **Read every
file** before editing — the layout IS the contract:

```
manifest.yaml                       # the whole pack is declared here (id, agent, flows, channels)
flows/tools/home.flow.yaml          # the MENU HUB — your front door (a list_picker; see the craft-ux guide)
flows/tools/home.locale.ar.yaml     # the hub's user-visible strings, keyed (referenced via $t)
flows/tools/browse.flow.yaml        # an example tool the hub invokes — replace with your real capability
flows/tools/browse.locale.ar.yaml   #   (+ its strings)
prompts/identity.md                 # the agent's system prompt (who it is, when to call tools)
pack.json                           # deploy target (platform_url, tenant, project) — not part of the pack
```

## Step 0.5 — Pull the capability reference (the source of truth, not this skill's prose)

Everything the platform supports — every built-in `do:` primitive, every expression builtin (`$…`), every
`render_intent`, the full flow-DSL schema, **the declaration schemas for what you write** (`manifest.yaml`,
`xrm.yaml`, `worklist.yaml`/casework, `scheduling.yaml`, …), **the platform's built-in system entities**
(`product`/`cart`/`order`/`case`/`booking`), the module authoring docs, **and the pack-authoring craft
guides** — is **not** in this skill. It lives in the **platform KB**, which you **pull straight from YOUR
platform** so it always matches that platform's exact version (this skill ships no bundled copy — a snapshot
only goes stale):

```bash
# one-time: point pack.json at your platform + `octwin login` (see Step 3), then:
octwin platform-kb pull        # → .octwin/platform-kb/  (markdown docs + *.json catalogs)
```

**Pull it before you author** and read it as you go — start with **`craft-capabilities`** (the orientation),
then the craft guides (`craft-ux`, `craft-flows`, `craft-manifest`, `craft-data-render`) and reference docs
(`primitives-guide`, `dsl`, `runtime`, …), and open the `*.json` catalogs for precise field schemas. The CLI
**watches it for drift**: after any command that talks to the platform, it prints a one-line nudge to re-run
`octwin platform-kb pull` whenever the platform's reference has changed since your last pull — so your copy
never silently goes stale.

**Consult it whenever you need to know whether a step / function / render-intent exists or its exact
fields.** **Never invent a built-in from memory or copy a list out of this skill's prose** — the outline
below is a map, not the reference; the KB is authoritative.

## Step 1 — Author the domain

> **Build a product, not a bag of tools.** A pack is a **guided product with a front door**. Three patterns
> make it one — every reference pack uses all three, and `octwin init` scaffolds them: a **home hub** (the
> menu the agent opens on a greeting / unclear message, wired FIRST in `flows` + the agent's `tools`),
> **rich interaction** (icons + a `carousel` for photo lists), and a **confirmation before any committing
> action** (the `approve_apply` preview → ✅/❌). How to build each well — with real YAML — is the
> **`craft-ux` guide** in the pulled KB; read it before you build the flows.

Four decisions, each with its authoritative guide in the pulled KB (`.octwin/platform-kb/`, Step 0.5):

- **Manifest (`manifest.yaml`)** — declares the whole pack (`id`, a **string** `version` — `'1.0.0'`, not a
  bare `1`; `description`, `supported_channels`, `required_adapters`, `default_settings.locale`, `flows`,
  `agents`; optional record modules via a sibling `xrm.yaml`/`worklist.yaml`). It is **strict** — unknown keys
  are rejected on deploy. Craft guide: **`craft-manifest`**; the exact schema of every declaration file: the
  **`declarations`** catalog.
- **Flows (`flows/tools/<flow>.flow.yaml` + `<flow>.locale.<lang>.yaml`)** — each agent-callable capability is
  a flow: `input:` (the tool's schema) → `entry:` (routing) → `steps:` (nodes). A `do:` calls a platform
  **built-in**; a `render:` node composes the UI. Three hard rules: all user-visible copy lives in the locale
  file (`$t("<flow>.<key>")`, **namespaced** — a bare/missing key fails validation); the **renderer owns
  channel limits** (never `.slice()`/clamp/page/read a cap in YAML — return all the data + one render intent;
  litmus: *would the value change if WhatsApp were swapped for web? then it's the renderer's job*); and the
  tool's **returned envelope** (`render`/`memory_note`) drives the next turn, not the prompt. Craft guide:
  **`craft-flows`**; the built-in + render-intent lists: the **`primitives`** / **`flow-schema`** catalogs.
- **Data — a first-class module, not a database.** Most packs need no DB: declare `xrm.yaml` (records ± a
  stage pipeline; also seeds demo data, optionally with AI images) or `worklist.yaml` (support tickets /
  casework) and get storage + the `record_*` / `case_*` built-ins for free. Craft guide:
  **`craft-data-render`** (+ the `xrm-guide` / `casework-guide` reference docs).
- **Agent (`prompts/identity.md` + `manifest.agents[]`)** — expresses **domain intent** and calls flow-tools;
  stays channel-agnostic (the tool decides the channel UI). The prompt says *when* to call a tool; the
  envelope controls *what happens next*. Lead titles/headers/buttons with an emoji (in the locale string, not
  the YAML). Keep it brief.

## Step 2 — Validate

```bash
octwin validate            # structure + pure-YAML rules, offline (fast, no server)
octwin validate --remote   # + the platform's FULL manifest + flow-DSL check — ALL errors at once
```

`octwin validate` is an offline structural pre-check. Once you've `octwin login`'d, run
**`octwin validate --remote`** to run the *exact* validation `deploy` runs (manifest `.strict()` + every flow's
schema/expression/structure) and get **every** error in one pass — so you fix them all before deploying instead
of one per failed deploy. (`octwin test` is an alias for `validate --remote`.)

## Step 3 — Deploy + test on your tenant

```bash
# one-time: point pack.json at your platform + tenant, then log in with a deploy token
#   (Octwin console → your workspace → API tokens → Generate)
octwin login --url https://your-octwin.example.com --token oct_…

octwin deploy --seed       # --seed also loads any demo data your xrm.yaml declares
octwin status              # "✓ live and current" once it's warm
# → chat with it on your tenant (web widget / console test page)
```

`deploy --seed` streams **live progress** — a line per demo record and per AI image as they're created
(image generation is the slow part), then a `Seeded: N record(s), M image(s)…` summary. Re-seeding is
idempotent and **reuses** already-generated images (it won't regenerate on every deploy).

Edit and `octwin deploy` again — a redeploy **hot-loads with no restart**; re-run `octwin status`
to confirm the live version caught up. A deploy is **durable**: the pack is stored server-side and
**reloaded automatically when the platform restarts** (it's warmed at boot alongside the built-in packs,
so the console lists it and it's chattable immediately). You never need to redeploy just because the
platform bounced.

### Debugging a live conversation from the CLI (octwin-cli ≥ 0.1.9)

The platform keeps **one open conversation per `--as` handle**, so consecutive `octwin chat` calls
with the same handle **continue the same conversation** — agent memory, suspended flows, taps and all.
Renders print in full (every list row / button / carousel card **with its tap id**), so you can drive
any tap-routed flow headlessly:

```bash
octwin chat "hi" --as t1                       # turn 1 — menu prints with each row's tap id
octwin chat --tap "t:invoke:my-flow:x=1" --as t1   # turn 2 — press a rendered row/button
octwin chat "free text answer" --as t1         # turn 3 — typed reply into the running flow
octwin logs --as t1                            # find the conversation (handle + last activity)
octwin logs <conversationId>                   # full timeline: taps, tool events, AND each turn's renders
octwin cases                                   # casework packs: the ticket inbox; `octwin cases <id>` = timeline
```

`--json` on `chat`/`logs`/`cases` dumps raw envelopes/events when you need exact payloads. `octwin
records <entity>` inspects XRM data (cases/tickets are casework — that's `octwin cases`, not `records`).

## Step 4 — Report your authoring experience (optional, encouraged)

Once you reach ✓ live — especially if it took several tries — turn what you learned into a **structured
feedback report** for the Octwin platform team. Every deploy that failed on something the local tooling
passed, every doc that was wrong, every hour lost is a fixable gap; a precise report is how it gets fixed.

**Write it to `FEEDBACK.md` in your pack directory** (the folder you're authoring in). Report only what you
actually hit — never invent friction. Cite the exact error text and the file/flow that triggered it. Group
each finding by the owner that can fix it. (An automatic `octwin feedback` submit path isn't wired yet — hand
`FEEDBACK.md` to the platform team.) Use this structure, deleting any owner section you have no findings for:

```markdown
## Octwin pack authoring feedback

- **Pack:** <id>@<version> — <one line: what the bot does>
- **Scope built:** <flows / modules, e.g. "7 flows; xrm + scheduling + casework; bilingual; media">
- **Platform:** <platform_url> · KB content_hash <from .octwin/platform-kb/index.json>
- **Deploy attempts to reach ✓ live:** <n> — <were the failures one-error-at-a-time?>

### A · CLI (octwin-cli)          # the local tool: validate / deploy / status behaviour, output, config
- **Finding:** <short title>
  - **Error / behaviour:** <exact message or what happened>
  - **Impact:** <what it cost — time, a false ✓, a silent failure>
  - **Fix:** <the concrete change you'd want>

### B · Platform                  # the server rejected what the CLI accepted, or a feature didn't load
- **Finding:** <short title>
  - **Error / behaviour:** <exact message — include the file/flow/field that triggered it>
  - **Local vs deploy:** <did `octwin validate` pass but the deploy reject? that gap is the finding>
  - **Fix:** <schema tweak, missing template, RBAC grant, …>

### C · Skill / KB                # the authoring guidance or capability reference was wrong/incomplete/opaque
- **Finding:** <short title>
  - **Gap:** <what the guidance/KB said vs. what was true, or what it didn't say at all>
  - **Fix:** <the doc/KB change — a value to surface, a shape to document, a sugar to version-gate>

### What went well                # genuinely — what saved time (scaffold, pulled KB, hot-reload, ports, pure-YAML)
- <...>

### Top asks (prioritized)        # the 3–5 changes that'd most help the next author, most-impactful first
1. <owner> — <the ask> — <why it's #1>
```

## Guardrails

- **No code.** `.ts`/`.js`, `routes/`, DB clients, and `*.primitives.*` are rejected on deploy. If a
  flow seems to need custom logic, express it with built-in steps + the expression grammar, or model
  the data with `xrm.yaml`. (Custom primitives are a platform-internal capability, not an external one.)
- **Don't invent structure** — scaffold with `octwin init` and edit. When unsure of a field, check the
  pulled KB rather than guessing; the manifest is strict and rejects unknown keys.
- **Locale everything.** A bare or missing `$t` key fails validation.
- **Renderer owns caps/paging/splitting.** Never read a channel limit in your YAML.
- **Version is a string.** `version: '1.0.0'` — a bare `1` is a number and fails.
