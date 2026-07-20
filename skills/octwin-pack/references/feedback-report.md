# Authoring-experience feedback report — template

Use this to turn a real authoring/deploy session into a **structured report for the Octwin platform
team**. The value is specifics: every finding is an *exact* error, its *impact*, and a *concrete fix*,
tagged with the layer that owns it — so the platform team can act without re-deriving what you hit.

**How to fill it in (for the authoring agent):** you lived through the session — you know which errors
the local tooling passed but the deploy rejected, which docs were wrong, and where you lost time. Write
from that. **Only include findings you actually hit** (never invent friction to fill the template). Cite
the exact error text and the file/flow that triggered it. Group each finding by owner:

- **A · CLI (`octwin-cli`)** — the local tool: `validate` / `deploy` / `status` behaviour, its output, its config handling.
- **B · Platform** — the server rejected something the CLI accepted, or a documented feature didn't load: schema mismatches, deploy-only validation, RBAC, missing runtime templates.
- **C · Skill / KB** — the authoring guidance or the capability reference was wrong, incomplete, or opaque: a primitive's `ok` payload wasn't documented, a render shape was unclear, sugar was version-gated without saying so.

Write the finished report to `FEEDBACK.md` in the pack directory. (Sharing it back to the platform
automatically — an `octwin feedback` submit command — isn't wired yet; for now hand `FEEDBACK.md` to the
platform team.)

---

## Octwin pack authoring feedback

- **Pack:** `<id>@<version>` — `<one line: what the bot does>`
- **Scope built:** `<flows / modules used, e.g. "7 flows; xrm + scheduling + casework; bilingual; media">`
- **Platform:** `<platform_url>` · KB reference version `<from octwin platform-kb pull, or "bundled snapshot">`
- **Date:** `<YYYY-MM-DD>`
- **Deploy attempts to reach ✓ live:** `<n>` — `<one line: were the failures one-error-at-a-time?>`

### A · CLI (`octwin-cli`)

> One block per finding. Delete the heading if you have none for this owner.

- **Finding:** `<short title>`
  - **Error / behaviour:** `<exact message or what happened>`
  - **Impact:** `<what it cost you — time, a false ✓, a silent failure>`
  - **Fix:** `<the concrete change you'd want>`

### B · Platform

- **Finding:** `<short title>`
  - **Error / behaviour:** `<exact message — include the file/flow/field that triggered it>`
  - **Local vs deploy:** `<did octwin validate pass but the deploy reject? that gap is the finding>`
  - **Fix:** `<the concrete change — schema tweak, missing template, RBAC grant, …>`

### C · Skill / KB

- **Finding:** `<short title>`
  - **Gap:** `<what the guidance/KB said vs. what was true, or what it didn't say at all>`
  - **Fix:** `<the doc/KB change — a value to surface, a shape to document, a sugar to version-gate>`

### What went well

> Genuinely — what saved you time or worked as promised (scaffold, bundled KB, hot-reload, the port/error
> model, pure-YAML expressiveness). This calibrates the team on what NOT to change.

- `<...>`

### Top asks (prioritized)

> The 3–5 changes that would most improve the next author's run, most-impactful first.

1. `<owner> — <the ask> — <why it's #1>`
2. `<...>`
