# octwin-pack — Claude Code plugin

> A **Claude Code plugin** whose skill teaches an AI agent to author pure-YAML **Octwin** packs and deploy them with [`octwin-cli`](https://www.npmjs.com/package/octwin-cli). By **CEQUENS**.

Octwin is a pack-pluggable conversational-agent platform for WhatsApp and web. A **pack** is a bot
domain — its agent, conversation flows, prompts, and data — declared entirely in YAML. Install this
plugin and your Claude Code gains the **`octwin-pack`** skill: it scaffolds a pack, authors the flows
and data model, grounds itself on your platform's live capability reference, validates, and deploys to
your tenant — all pure YAML, no TypeScript.

> Companion CLI: [`octwin-cli`](https://www.npmjs.com/package/octwin-cli) (`npx octwin-cli`). The skill drives it.

## Install (Claude Code plugin)

This is distributed as a **Claude Code plugin via a marketplace** (not npm). In Claude Code:

```
/plugin marketplace add <owner>/<repo>      # the git repo hosting this plugin
/plugin install octwin-pack@octwin           # plugin "octwin-pack" from the "octwin" marketplace
```

Then just ask, e.g.:

> "Scaffold an Octwin pack for a dental clinic that lists services and books appointments."

The skill scaffolds with `octwin init`, pulls the platform capability reference (`octwin platform-kb
pull`), authors `manifest.yaml` / flows / prompts / `xrm.yaml`, validates as it goes, and walks you
through `octwin login` / `octwin deploy` to test live on your tenant.

## What's in here

```
.claude-plugin/
  plugin.json          # plugin manifest
  marketplace.json     # marketplace listing (source: this repo)
skills/octwin-pack/
  SKILL.md             # the authoring skill
  references/
    manifest.md · flows.md · data-and-render.md   # the CRAFT layer (how to build well)
    capabilities.md                                # index into the platform KB (what exists)
    platform-kb/                                   # bundled KB snapshot (offline fallback)
```

The skill treats the **platform KB** as its source of truth for what the platform supports (built-in
`do:` primitives, expression builtins, render intents, the flow-DSL). It reads the **live** reference
from `.octwin/platform-kb/` after `octwin platform-kb pull`, and falls back to the committed snapshot in
`skills/octwin-pack/references/platform-kb/`.

## Requirements

- **Claude Code** + **Node.js ≥ 20**
- An **Octwin platform**, a **tenant**, and a **deploy token** (for pull/deploy) — see the
  [`octwin-cli`](https://www.npmjs.com/package/octwin-cli) docs.

## Maintainers

The bundled KB snapshot is regenerated from the platform with `npm run sync:plugin-kb` (after
`npm run platform-kb`) and committed. Re-sync when the platform stdlib changes.

## License

[MIT](./LICENSE) © CEQUENS
