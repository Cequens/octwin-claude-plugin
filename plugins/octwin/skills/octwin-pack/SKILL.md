---
name: octwin-pack
description: Author a pure-YAML Octwin pack — a WhatsApp/web conversational bot declared entirely in YAML — and deploy it to your tenant with octwin-cli. Use when the user wants to build, scaffold, or extend an Octwin pack (its agent, conversation flows, and data) for their own business, with no platform checkout and no TypeScript.
license: MIT
allowed-tools: Bash(octwin *) PowerShell(octwin *)
---

# Authoring an Octwin pack (pure-YAML)

An **Octwin pack** is a self-contained conversational bot for WhatsApp + web — its agent, its
conversation **flows**, its prompts, and its data model — declared **entirely in YAML**. You build it
in your own repo and deploy it to a running Octwin platform with the
[`octwin`](https://www.npmjs.com/package/octwin-cli) CLI; the platform runs it on your tenant.

**This skill is the workflow, not the reference.** It takes you from nothing to a deployed, chattable
pack. Every capability, field, schema, example and gotcha — and every guide that teaches them — lives
in the **platform KB**, pulled from your own platform in Step 0.5. It has to match that platform's
exact version, and anything copied into this file would silently go stale, so this file names no
guide, no capability and no count. It tells you *when* to read the KB; the KB's own index tells you
*what* to read.

Two rules frame everything:

- **Pure declarative data only.** A pack is `.yaml` / `.md` / `.json` plus pictures — **no `.ts`/`.js`,
  no `.sql`, no custom code, no custom primitives**. Flows compose the platform's **built-in** steps;
  your data model is declared in `xrm.yaml` (a pack owns no database). This is what lets the platform
  safely run your pack, and the server enforces it on every deploy.
- **Scaffold, then customize.** Never hand-write pack boilerplate from memory — start from
  `octwin init` and edit. The scaffold is a real, valid, chattable hello-bot.

## Step 0 — Install the CLI and scaffold

```bash
npm i -g octwin-cli        # install once; then `octwin <cmd>` everywhere below
octwin --version           # confirm it's on the current release
# No-install fallback (CI, or can't install globally): npx octwin-cli@latest <cmd>

octwin init ./my-pack --id my-pack --description "What the bot does"
cd ./my-pack
```

This writes a minimal, valid, chattable pack that already models the **home-hub** shape. **Read every
file before editing — the layout IS the contract**, and the scaffold answers most "where does this
go?" questions before you have to ask them. The directory is pack content only; your deploy target
lives in your saved login (Step 3), never in a file here.

## Step 0.5 — Pull the capability reference

**Do this before you author anything.** Everything the platform supports — every built-in `do:` step,
every `$…` expression function, every render intent, the schema of every file you write, the built-in
entities you get free, the module guides, and the pack-authoring **craft guides** — is in the KB, not
in this skill:

```bash
octwin platform-kb pull --if-stale --url https://your-octwin.example.com   # → .octwin/platform-kb/
```

**No token needed** — the reference is platform stdlib and is served anonymously, so this works on a
fresh clone before you have logged in. **Run it at the start of every session:** `--if-stale` checks
the platform's content hash first and does nothing when your copy is current, so it costs one small
request. **Pull at your repo ROOT** — `octwin validate` walks *up* from the pack directory, so one
pull covers every pack in a monorepo. A set of `validate`'s checks is driven by this reference and
**skips without it** — the run says how many on its last line, and `--require-kb` makes that a
failure for CI.

**Open `INDEX.md` first.** The pull writes its own maps beside the reference, and `INDEX.md` is the
front door: it names every doc with its size, groups them by family, gives a reading route, and points
you at the right map for the other two questions — a bare name you need to resolve, or one subject
inside a heavy doc. Read it before you read anything else, and never read a whole catalog to find one
entry.

**If it isn't in the KB, it doesn't exist for a pure-YAML pack.** Never invent a built-in, a field or
a render-intent key from memory, and never infer one intent's fields from another's — the platform
rejects unknown keys and `octwin validate` flags them offline. The CLI nudges you (on stderr, even
when output is piped) whenever the platform's reference changes, so your copy never goes quietly
stale; `octwin platform-kb --check` reports the same thing as an exit code (0 current, 2 stale).

## Step 1 — Author the domain

> **Build a product, not a bag of tools.** A pack is a **guided product with a front door**. Three
> patterns make it one — every reference pack uses all three, and `octwin init` scaffolds them: a
> **home hub** (the menu the agent opens on a greeting / unclear message, wired FIRST in `flows` + the
> agent's `tools`), **rich interaction** (icons + a `carousel` for photo lists), and a
> **confirmation before any committing action**. Read the KB's UX guide before you build the flows.

Every decision you are about to make — what the pack declares, how a flow is written, how records are
stored and rendered, what happens when a fetch returns nothing or too much, what happens when a record
moves without a flow touching it, how the agent is instructed, how you reach a customer who isn't
typing, and what happens when the bot can't help — has **one authoritative guide** in the pulled KB
(`.octwin/platform-kb/`). Capability-specific ones (bookings, selling, another system, campaigns) sit
beside them.

Read the guide rather than guessing from here — that is the whole point of pulling it. `INDEX.md`
names them all with a reading route: read the core set in order, then add the guide for each
capability your business actually needs, and skip the rest. Reading the whole set is not the plan.

The agent expresses **domain intent** and calls flow-tools; it stays channel-agnostic, because the tool
decides the channel UI. The prompt says *when* to call a tool; the tool's returned envelope controls what
happens next. Keep it brief.

## Step 2 — Validate

```bash
octwin validate            # offline: structure, the pure-YAML rules, + the checks driven by the pulled KB
octwin validate --remote   # the platform's FULL check — every error in ONE pass
```

Run **both**, in that order. The offline run replays the platform's own contracts from the reference
you pulled in Step 0.5 — render-intent fields, primitive arguments, `$function()` names, `use:`
template params, reserved entity keys, and the declaration files' schemas — with no server and no
token. Add `--require-kb` in CI so a missing reference fails the build instead of quietly skipping
those checks. `--remote` runs the *exact* validation `deploy` runs, so it turns eight
fix-one-error-and-redeploy cycles into one. (`octwin test` is an alias for it.) What its
project-resolution errors mean is in the KB — `INDEX.md`'s symptom table routes you to the guide.

## Step 3 — Deploy and drive it

```bash
# one-time (console → your workspace → API tokens → Generate):
octwin login --url https://your-octwin.example.com --token oct_…

octwin projects            # the --project slugs this token can name
octwin deploy --seed       # --seed also loads the demo data xrm.yaml declares
octwin status              # "✓ live and current" once it's warm
```

Then drive it from the terminal — `octwin chat` prints every render **with its tap id**, so you can
press buttons headlessly, and `octwin logs <conversationId>` shows the full turn timeline.

**For a multi-step conversation use `octwin chat --script <file>`, never chained `chat` calls** —
chaining races the agent loop and produces a timing bug that reads as a flow bug. That, the read-back
commands (`records` / `orders` / `analytics` / `agents --prompt` / `scheduling --slots`), the two
things that look like a broken pack and are not (a second install on a project that already
dispatches elsewhere; a redeploy invalidating every **suspended** run), and disposable test projects
are all in the KB's testing guide.

Before you call it done, make the pre-ship pass at the end of that same testing guide — a dozen
defects that pass every gate, deploy cleanly, and are visible only to the customer. `validate` cannot
see any of them.

## Step 4 — Report what you learned, and read what comes back

Once you reach ✓ live — especially if it took several tries — write `FEEDBACK.md` in your pack
directory and send it with **`octwin feedback`**. Report only what you actually hit, cite the exact
error text, and group each finding by the owner who can fix it. The template and the reporting rules
live in the KB's feedback guide.

**The channel answers, so read it.** When any command prints `✉ N memos from the platform`, run
**`octwin memos`** and act on them **before continuing** — do not finish the task first. Two kinds
arrive: a reply to a report you sent, and a notice published to every author (a new capability, a
deprecation, a breaking change). This matters because the thing you are about to work around may
already be fixed: in August 2026 an author hardcoded English across 16 flows to avoid a `$t()` bug
that had been fixed two days earlier, because nothing could tell them. Reading marks them read, so
the reminder stops; `octwin memos --all` re-reads history.

## Guardrails

- **No code.** `.ts`/`.js`, `routes/`, DB clients and `*.primitives.*` are rejected on deploy. If a
  flow seems to need custom logic, express it with built-in steps + the expression grammar, or model
  it as data. A genuine gap in the grammar is a **platform** gap — report it (Step 4) rather than
  looking for a bypass, because there isn't one.
- **Resolve every platform name through `SYMBOLS.md` before you write it.** A `do:` step, an `args:`
  key, a `$function()`, a render-intent field, a node op-key, a key in a declaration file — grep the
  name, open the file the row points at, write what it says. This is the procedure behind "if it
  isn't in the KB it doesn't exist": a name absent from that map does not exist, and the reason to
  grep rather than recall is that an invented name reads exactly like a real one.
- **Don't invent structure.** Scaffold with `octwin init`; check the pulled KB rather than guessing.
- **Locale everything.** All user-visible copy lives in the locale file; a bare or missing `$t` key
  fails validation.
- **The renderer owns caps, paging and splitting.** Never clamp, slice or read a channel limit in your
  YAML. Litmus: *would this value change if WhatsApp were swapped for web?* Then it isn't yours.
- **Version is a string.** `version: '1.0.0'` — a bare `1` is a number and fails.
