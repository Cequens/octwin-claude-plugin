# Changelog — octwin-pack-author (Claude Code plugin)

All notable changes to the `octwin-pack-author` plugin — the **`octwin-pack`** authoring skill.
Format: [Keep a Changelog](https://keepachangelog.com/) — newest first.
The platform-wide view lives in the repo root [`CHANGELOG.md`](../../CHANGELOG.md); the companion CLI
has its own [`packages/octwin-cli/CHANGELOG.md`](../octwin-cli/CHANGELOG.md).

## [Unreleased]

The plugin is now a **single `SKILL.md`** — no `references/` folder. Everything authoritative is pulled
from the platform KB; the plugin carries only the workflow.

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
