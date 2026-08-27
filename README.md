# octwin — Claude Code plugin

> A **Claude Code plugin** whose skill teaches an AI agent to author pure-YAML **Octwin** packs and
> deploy them with [`octwin-cli`](https://www.npmjs.com/package/octwin-cli). By **CEQUENS**.

Octwin is a pack-pluggable conversational-agent platform for WhatsApp and web. A **pack** is a bot
domain — its agent, conversation flows, prompts, and data — declared entirely in YAML. Install this
plugin and your Claude Code gains the **`octwin-pack`** skill: it scaffolds a pack, authors the flows
and data model, grounds itself on your platform's live capability reference, validates, and deploys to
your tenant — all pure YAML, no TypeScript.

> Companion CLI: [`octwin-cli`](https://www.npmjs.com/package/octwin-cli) — install once with
> `npm i -g octwin-cli` (or `npx octwin-cli@latest` to run without installing). The skill drives it.

## Install

Distributed as a **Claude Code plugin via a marketplace** (not npm). In Claude Code:

```
/plugin marketplace add Cequens/octwin-claude-plugin    # the git repo hosting this plugin
/plugin install octwin@octwin-claude-marketplace        # plugin "octwin" (bundles the octwin-pack skill)
```

**Read the line the install prints.** It says either `Plugin is now active.` or
`Run /reload-plugins to activate.` — in the second case the skill is not loaded until you do.

Then just ask, e.g.:

> "Scaffold an Octwin pack for a dental clinic that lists services and books appointments."

Claude loads the skill on its own when the request matches. To invoke it by hand, either form works:

```
/octwin:octwin-pack     # namespaced by plugin
/octwin-pack            # bare — works unless another command claims the name
```

The skill scaffolds with `octwin init`, pulls the platform capability reference (`octwin platform-kb
pull`), authors `manifest.yaml` / flows / prompts / `xrm.yaml`, validates as it goes, and walks you
through `octwin login` / `octwin deploy` to test live on your tenant.

### Staying current

This plugin releases often. To pull the latest without waiting on background refresh:

```
/plugin marketplace update octwin-claude-marketplace
```

The marketplace name (`octwin-claude-marketplace`) is **not** the repo slug, and it is the name you
need for updating. You can also manage auto-update per marketplace under `/plugin` → Marketplaces.

> ⚠️ Use `update`, never `remove`. Removing a marketplace from its last remaining scope
> **uninstalls every plugin you installed from it.**

> **Upgrading from `octwin-pack-author`?** That was the plugin's name before 0.2.0. Claude Code has no
> rename mechanism, so the old name is a *different* plugin. Install `octwin` above, then remove the
> old entry **by hand** — the obvious command does not work:
>
> ```
> claude plugin uninstall octwin-pack-author@octwin-claude-marketplace
> → ✘ Plugin "octwin-pack-author@octwin-claude-marketplace" not found in installed plugins
> ```
>
> Once the marketplace stops listing that name, `uninstall` can no longer resolve it — while
> `claude plugin list` still shows it **enabled** and its cached skill still loads. Measured
> 2026-08-27. Delete its line from the `enabledPlugins` block of `~/.claude/settings.json` instead:
>
> ```json
> "octwin-pack-author@octwin-claude-marketplace": true,
> ```
>
> The skill and its commands are otherwise unchanged.

## What this plugin does on your machine

Worth knowing before you install anything from a repo you don't own:

- **Installs a CLI globally** — the skill's first step is `npm i -g octwin-cli`. This is left to prompt
  for your approval on purpose.
- **Writes inside your project only** — the pack you are authoring, plus a `.octwin/` directory
  holding the pulled capability reference and your saved login.
- **Talks to one host: the platform URL you supply.** There is no telemetry endpoint and no other
  network destination. `octwin feedback` sends a report to that same platform, and only when you run it.
- **Ships no executable code.** The plugin is one Markdown file.

### Fewer permission prompts

The skill drives `octwin` across many turns. Its frontmatter pre-approves that command, but a
frontmatter grant covers only the turn that invokes the skill and clears on your next message. For a
whole authoring session, add an allow rule to your own permission settings:

```json
{ "permissions": { "allow": ["Bash(octwin *)", "PowerShell(octwin *)"] } }
```

Both entries are deliberate — `Bash` and `PowerShell` are separate tools with separate rules, and
which one Claude Code uses depends on your platform. `npm i -g` is intentionally left out.

## Requirements

- **Claude Code** + **Node.js ≥ 20**
- An **Octwin platform**, a **tenant**, and a **deploy token** (for pull/deploy) — see the
  [`octwin-cli`](https://www.npmjs.com/package/octwin-cli) docs.

## What's in here

```
.claude-plugin/marketplace.json   # marketplace listing
.github/workflows/validate.yml    # CI — both validator passes
plugins/octwin/                   # the plugin payload — this, and only this, reaches users
  .claude-plugin/plugin.json      #   plugin manifest
  skills/octwin-pack/SKILL.md     #   the authoring skill — the whole plugin
  CHANGELOG.md · LICENSE
```

The skill is a single `SKILL.md`, and it is deliberately **thin**: the init → pull → author →
validate → deploy → feedback workflow, and nothing else. Every capability the platform supports and
every pack-authoring craft guide lives in the **platform KB**, read from `.octwin/platform-kb/` after
`octwin platform-kb pull` — fetched live from your own platform, so it always matches that platform's
version. `INDEX.md` in the pulled reference names every guide.

The plugin ships **no bundled snapshot and no list of what the KB contains.** The KB is republished
far more often than this plugin, so anything copied here would be stale between releases. The CLI
instead nudges you to re-pull whenever the platform's reference changes.

## Maintainers

Update + publish in **one command** from the platform monorepo:

```bash
npm run plugin:publish                # mirror → this repo, validate, bump + push
npm run plugin:publish -- --dry-run   # preview the diff without pushing
```

A clean **no-op when nothing changed**, so it is safe to run any time. Run it whenever `SKILL.md`
changes. Platform-stdlib and KB changes alone need **no** republish — that is the point of the split.

## License

[MIT](./plugins/octwin/LICENSE) © CEQUENS
