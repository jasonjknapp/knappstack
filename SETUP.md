# Setup — adopting `knappstack` in your project

`knappstack` is a **playbook + templates**, not a one-click installer. You adopt it by *instantiating* it into your project — tailored to your stack — and the fastest way is to let your coding agent do that work for you.

---

## The fast way (AI-first)

Open your coding agent (e.g. Claude Code) **inside your project**, make sure it can see this repo (clone it locally, or have it shared with you — see [Access](#access)), and give it a prompt like:

> Read the `knappstack` repo at `<local path or repo URL>`. Set up this operating model in **this** project:
> 1. Create a `CLAUDE.md` from `templates/CLAUDE.md`, filled in for our stack and constraints — but **do not overwrite an existing `CLAUDE.md`**; show me a diff to merge.
> 2. Add `PLAN.md` and `QA_CASES.md` from the templates.
> 3. Treat `principles/ENGINEERING-PRINCIPLES.md` as standing instructions and link it from our `CLAUDE.md`.
> 4. Wire the gates in `release-and-review/` to our actual CI, error tracker, and PR-review bot.
> 5. Propose a few **stack-specific** slash commands / skills that implement the lifecycle and the release gate, and create them under `.claude/` once I approve.
>
> Explain what you changed and why. Commit nothing until I review.

Review the diff, then commit. The agent reads the doctrine and builds *your* tailored version — that's the point. It's better than copying someone else's skill files, because the result fits your stack instead of a stranger's.

## What you end up with

- **`CLAUDE.md`** — your project constitution (hard rules, always-current facts, conventions).
- **Plan-state convention** — the `PLAN.md` header + durable-planning rules so multi-session work doesn't drift.
- **`QA_CASES.md`** — the living, only-grows QA list.
- **A review + release gate** — cross-model review + a staged, human-authorized release path, wired to *your* tools.
- **(Optional) skills/commands** — stack-specific ones your agent authors from the playbook.

## Prefer to do it by hand?

1. Copy `templates/CLAUDE.md`, `templates/PLAN.md`, `templates/QA_CASES.md` into your repo; fill every `<placeholder>`.
2. Make `principles/ENGINEERING-PRINCIPLES.md` your agent's standing instructions (link it from `CLAUDE.md`).
3. Read `lifecycle/`, `release-and-review/`, and `planning/`, and wire the gates to your CI / error tracker / PR-review bot.

## Customize for your world

- **Stack** — swap the generic examples for your languages, CI runner, hosting target, and error tracker.
- **Team vs. solo** — the gates apply either way; only *how* they map differs. The canonical breakdown lives in [`release-and-review/STAGED-RELEASE.md`](release-and-review/STAGED-RELEASE.md).

## Access

While this repo is private, point your agent at a **local clone** (or get collaborator access first) — an agent can't read a private repo from a URL without your credentials. Once it's public, the URL works directly.

## Keep it in sync

It's a living playbook. `git pull` for updates and re-run the relevant setup step. The principles change rarely; the tool-wiring in `release-and-review/` is what you'll revisit as your stack evolves.
