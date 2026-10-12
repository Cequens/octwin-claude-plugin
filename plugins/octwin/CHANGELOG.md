# Changelog — octwin (Claude Code plugin)

All notable changes to the `octwin` plugin — the **`octwin-pack`** authoring skill.
Format: [Keep a Changelog](https://keepachangelog.com/) — newest first.
The companion CLI publishes its own release notes on
[npm](https://www.npmjs.com/package/octwin-cli).

## [0.2.6] - 2026-10-12

### Changed
- **The skill says what `octwin init` gives you.** It starts you with the home hub — the first of the
  three patterns — not all three; the others come from the KB's recipes. The plugin README asks for
  Node 22 (what `octwin-cli` requires), says a workspace is enough to start (pulling and validating
  need no login; deploying signs you in through the browser), and names both places the CLI writes:
  your project's `.octwin/` and `~/.octwin/` in your home folder.
- **The skill now calls what you build an AI Agent.** Five lines, the description among them
  — it keeps "chatbot" so the skill is still found by the word a developer types. The pulled KB it
  points at made the same change in 41 places, and the platform's brand gate now reads both, so
  the word cannot drift back.
- **Step 0 says to delete the scaffold files you do not keep.** A scaffold flow left behind with its
  `flows:` line removed never loads, but its copy is still checked — a pack that adds a second language
  then fails validation over a flow nobody uses. Found when an AI author built a clinic pack from the
  scaffold and left the example flow in place.
- **A pack that is already live starts from `octwin pull`, not a scaffold.** A short step after Step 0:
  pull the version your project runs, follow the pulled KB's route for changing a live agent before
  touching a declaration, and deploy under a bumped `version:` without `--seed`. Some changes that
  validate cleanly still break a live pack's records, conversations in progress or old buttons.
- **Step 2 is one command.** `octwin validate` runs the platform's full check by default and needs no
  login for it; the skill now tells you to read the LAST line, which says *PARTIAL* when only the
  offline half could run.

## [0.2.5] - 2026-09-06

### Changed
- **Step 3 signs in through the browser — you no longer paste a token to the agent.**
  `octwin login --url <your platform>` now prints an approval link instead of asking for a key you
  minted by hand. A workspace admin opens it, checks which machine is asking, and clicks Approve;
  the CLI collects a 30-day token by itself.

  This is why the step changed, and the skill now says it outright: the old instruction ended with a
  human pasting a live `oct_…` into a conversation — a workspace credential, sitting in a
  transcript. The agent's part is now to relay a public URL, wait to be told it was approved, and
  run `octwin login` again.

  The skill also states the exit codes, because an agent should branch on them rather than read
  prose: `0` signed in, `75` still waiting (the wait is bounded at 90s and resumes on the same
  request, which lives ten minutes), `1` failed.

  Needs **octwin-cli 0.8.8 or newer**. `--token` still works unchanged, and CI should still skip
  login and set `PACK_PLATFORM_URL` + `PACK_TOKEN`.

## [0.2.4] - 2026-08-27

### Fixed
- **The update instructions stopped one command short.** `/plugin marketplace update` refreshes the
  catalog but does **not** move your install: measured on a real upgrade, the plugin stayed on the
  previous version until `/plugin update octwin@octwin-claude-marketplace` ran. The README now gives
  both, and says plainly not to rely on auto-update — on the maintainer's own machine the plugin was
  six releases behind what was published.

## [0.2.3] - 2026-08-27

### Fixed
- **The upgrade note told you to run a command that does not work.** Renaming a plugin *strands* the
  old install: once the marketplace catalog stops listing `octwin-pack-author`,
  `claude plugin uninstall octwin-pack-author@octwin-claude-marketplace` answers
  `not found in installed plugins`, while `claude plugin list` still shows it **enabled** and its
  cached skill still loads. Measured on a real install. The README now says to delete the entry from
  `enabledPlugins` in `~/.claude/settings.json` by hand.

## [0.2.2] - 2026-08-27

> Supersedes 0.2.1, which shipped this same content minutes earlier under a version its own
> changelog heading did not name: the publish script read the published version from the *new*
> payload path, found nothing there because the move had not landed yet, and bumped a patch it
> should not have. Fixed in the script — with no published manifest to compare against, it now
> ships the version the source declares.

### Changed
- **BREAKING — the plugin is renamed `octwin`** (formerly `octwin-pack-author`). Claude Code has no
  rename mechanism, so the old name is a *different* plugin: uninstall it and run
  `/plugin install octwin@octwin-claude-marketplace`. The marketplace name, the repo, the skill name
  and both commands are unchanged; the namespaced form is now `/octwin:octwin-pack`, and the bare
  `/octwin-pack` still works.
- **The skill names no KB guide, no capability and no count.** The KB is republished far more often
  than this plugin, so every enumeration inside the plugin was stale between releases — the README
  had drifted to 6 guides against the skill's 17, and claimed a template was inlined that had moved
  to the KB. `SKILL.md` now routes through `INDEX.md` in the pulled reference and holds only the
  workflow: the two framing rules, the `octwin` command sequence, and the guardrails. Adding a guide
  to the KB no longer requires publishing this plugin.
- **`homepage` points at this repo**, not the CLI's npm page — Claude Code sends users to `homepage`
  for usage guidance, and they were landing on the wrong docs.

### Added
- **`license` and `allowed-tools` in the skill frontmatter.** `allowed-tools: Bash(octwin *)
  PowerShell(octwin *)` pre-approves the CLI on both shells — they are separate tools with separate
  permission rules, and PowerShell is the default on Windows without Git Bash. `npm i -g` still
  prompts, deliberately. Both fields are in the Agent Skills spec, so the file stays portable.
- **CI in this repo** — `claude plugin validate` over the marketplace catalog and the plugin
  manifest, on every push.
- **A "what this plugin does on your machine" section** in the README, plus post-install activation,
  the marketplace-update command, and the warning that removing a marketplace uninstalls its plugins.

### Fixed
- **Links that only broke once published.** The README and this file linked to the platform monorepo
  with paths that resolve there and 404 here. The publish script now refuses to mirror a link that
  escapes the plugin repo.

### Removed
- **The repo root is no longer the plugin payload.** The plugin moved to `plugins/octwin/`, so the
  marketplace catalog, CI and the repo README stop being copied into every user's install cache.

## [0.1.12] - 2026-08-09

### Changed
- **SKILL.md is now a shell: the workflow plus a map of which KB guide answers each decision.** All
  reference prose (the scaffold file listing, the KB directory layout, the debugging walkthrough,
  the four-decision narratives) moved into the platform KB's craft guides, where it versions with
  the author's own platform; the skill's Step 1 is now a decision table naming the guide per
  decision (`craft-manifest` / `craft-flows` / `craft-data-render` / `craft-result-states` /
  `craft-reactions` / `craft-agent` / `craft-outreach` / `craft-escalation` / `craft-booking` /
  `craft-commerce` / `craft-integrations` / `craft-campaigns` / `craft-testing` /
  `craft-principles`).
- **Step 0.5 teaches the anonymous, session-start pull.** The KB is served without a token, so
  `octwin platform-kb pull --if-stale --url <platform>` works on a fresh clone before login; the
  skill now says to run it at the start of every session (`--if-stale` no-ops when current) and
  documents the three lookup maps — `INDEX.md` (a question), `SYMBOLS.md` (a name), `OUTLINE.md`
  (a heading inside a heavy doc) — plus `octwin platform-kb --check` as the exit-code form.
- **Testing guidance routes through `craft-testing`.** The inlined chat/logs debugging section is
  replaced by the hard rules that stay in the skill — use `octwin chat --script <file>` for
  multi-step conversations (chained `chat` calls race the agent loop), and make the pre-ship pass
  before calling a pack done — with everything else (read-back commands, the two
  looks-broken-but-isn't cases, disposable test projects) read from the guide.

## [0.1.11] - 2026-08-01

### Changed
- **Step 2 says that `validate --remote` targets a project.** The skill introduced `--project` and
  `octwin projects` only in Step 3 (deploy), so an author with an unpinned token met
  `project required` at Step 2 having never been told the flag existed — the endpoint resolves tenant
  and project **before** validating, so `validate --project <typo>` fails wherever `deploy --project
  <typo>` would. Step 2 now states that, notes the checks themselves are project-blind, and names
  both failure messages so a flag problem is not mistaken for a pack problem.

## [0.1.9] - 2026-07-29

The plugin is now a **single `SKILL.md`** — no `references/` folder. Everything authoritative is pulled
from the platform KB; the plugin carries only the workflow. (The `references/` removal below shipped
progressively across 0.1.6–0.1.9, which went out without their own changelog cuts.)

### Changed
- **SKILL.md Step 0.5 teaches the exploded KB layout.** `octwin platform-kb pull` now writes an
  `INDEX.md` plus one file per capability (`primitives/record_list.json`,
  `render-intents/carousel.json`, …) instead of 34 flat catalogs, so the skill instructs the agent to
  read `INDEX.md` first and open only the entry it needs — never a whole catalog. Render intents get a
  named warning: each takes a **fixed** field set, the platform rejects anything outside it, and
  `octwin validate` flags it offline, so read `render-intents/<name>.json` before composing a card.
- **SKILL.md pack layout lists `commands.yaml`, `locale.ar.yaml`, and `messages.ar.yaml`** — the
  scaffold ships them, and the skill's "the layout IS the contract" file map had omitted all three.
- **`pack.json` is gone from the skill's workflow** — the saved `octwin login` is the deploy target.

### Removed
- **The entire `skills/octwin-pack/references/` folder.**
  - The **bundled platform-KB snapshot** (`references/platform-kb/`) — a committed copy of the reference
    only drifts from the platform it describes. `octwin platform-kb pull` is now the single source of
    truth (the live reference in `.octwin/platform-kb/`), and octwin-cli ≥ 0.1.10 nudges the author to
    re-pull when the platform's reference changes. `plugin:publish` no longer regenerates catalogs or
    syncs a snapshot — it's a pure mirror of the skill source.
  - The **craft docs** (`manifest.md` · `flows.md` · `data-and-render.md` · `ux-patterns.md` ·
    `capabilities.md`) **moved into the platform KB** as pulled docs (`craft-manifest` / `craft-flows` /
    `craft-data-render` / `craft-ux` / `craft-capabilities`), so the "how to build well" guidance lives
    beside the reference it grounds on and is pulled with it.
  - The **`feedback-report.md` template** is now inlined into SKILL.md Step 4 (generated straight into the
    author's working `FEEDBACK.md`).

### Changed
- **SKILL.md** rewritten around pull-only: Step 0.5 pulls the reference (craft guides + reference docs +
  catalogs) before authoring and re-pulls on the CLI's drift nudge; Step 1 is a lean four-decision map that
  points at the pulled craft guides; the FEEDBACK.md template is inline in Step 4.

## [0.1.5] - 2026-07-21

Driven by the second author-feedback round (xpeng-egypt): the two KB gaps it hit are closed, and the
skill now teaches the CLI's new conversation-debugging loop.

### Added
- **SKILL.md — "Debugging a live conversation from the CLI"** (Step 3): multi-turn `octwin chat`
  (one open conversation per `--as` handle), pressing rendered buttons with `--tap "<tap-id>"`,
  reading timelines with `octwin logs`, inspecting tickets with `octwin cases`, and `--json` payload
  dumps. Requires octwin-cli ≥ 0.1.9.

### Changed
- **Bundled platform-KB snapshot resynced**, picking up two catalog fixes the feedback round asked for:
  - `builtins.json` now indexes the **runtime-bound `$…` functions** (`$t`, `$t_has`, `$enum_label`,
    `$loc`, `$record_url`, `$work_types`, `$partial`, `$diff_lines`) with `kind: "runtime-bound"` —
    they previously existed only in dsl.md prose, so a catalog-first lookup read them as nonexistent.
  - `channels.json` now publishes each channel's **concrete `$caps` values** (list/buttons/carousel —
    including the previously-invisible `carousel.buttonsPerCard: 2` per-card button cap — plus text
    length limits and commerce caps). It was an empty `channels: []` before.
  - `dsl.md`'s `$caps` row fixed — it named non-existent fields (`buttons.maxRows`/`carousel.maxCards`;
    real: `buttons.max`/`carousel.max`) and now points at the catalog instead of re-spelling values.

## [0.1.3] - 2026-07-21
- Recommend the global CLI install (`npm i -g octwin-cli`) so the upgrade notice is actionable;
  `npx octwin-cli@latest` kept as the no-install fallback.

## [0.1.2] - 2026-07-21
- **UX patterns layer**: new `references/ux-patterns.md` — the home-flow navigation hub, rich
  interaction (icons, carousel images), and `approve_apply` confirmations — surfaced in SKILL.md
  Step 1 + an `on_select` placement table in `flows.md`.

## [0.1.1] - 2026-07-21
- KB snapshot resync (chore).

## [0.1.0] - 2026-07-20
- Initial plugin: the `octwin-pack` authoring skill (scaffold → author → validate → deploy with
  octwin-cli), craft references (manifest/flows/data-and-render), the feedback-report template, and
  a bundled platform-KB snapshot as the offline fallback for `octwin platform-kb pull`.
