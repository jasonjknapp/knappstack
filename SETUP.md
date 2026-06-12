# Setup — adopting `knappstack` in your project

`knappstack` is a **command surface + playbook + templates**, not a one-click installer. You adopt it two ways, which compose: **install the commands** (`workflows/` + `skills/`) so your agent can run them, and **instantiate the doctrine** into your project — tailored to your stack. The fastest path to the second is to let your coding agent do that work for you.

---

## Install the command surface

The commands live in two directories:

- **`workflows/`** — long-form `/`-style command runbooks (`engineering-principles`, `release-prep`, `release-prod`, `checkpoint`).
- **`skills/`** — model-invoked skills, one directory per skill, each with a `SKILL.md` (`brainstorm`, `plan`, `principal-consultant`, `release-check`, `peer-review`, `debug`).

Install them into your agent-config directory so they're available across projects. With Claude Code that's `~/.claude/commands` and `~/.claude/skills`. Symlink rather than copy so a `git pull` updates the live commands in place:

```sh
# from a local clone of this repo, e.g. ~/src/knappstack
mkdir -p ~/.claude/commands ~/.claude/skills

# workflows → commands (one symlink per file)
for f in ~/src/knappstack/workflows/*.md; do
  ln -sf "$f" ~/.claude/commands/"$(basename "$f")"
done

# skills → skills (one symlink per skill directory)
for d in ~/src/knappstack/skills/*/; do
  ln -sf "${d%/}" ~/.claude/skills/"$(basename "$d")"
done
```

Adjust the target paths for your harness — any agent that loads slash-commands from a config directory follows the same pattern. (If your config directory lives on a synced drive, symlinks are safer than copies anyway: one source of truth, no copy to drift.)

Frontmatter convention, if you author your own:

- **Skills** open with `---\nname: <name>\ndescription: <one line>\n---`.
- **Workflows / commands** open with `---\ndescription: <one line>\n---`.

---

## The fast way (AI-first)

Open your coding agent (e.g. Claude Code) **inside your project**, make sure it can see this repo (clone it locally, or have it shared with you — see [Access](#access)), and give it a prompt like:

> Read the `knappstack` repo at `<local path or repo URL>`. Set up this operating model in **this** project:
> 1. Create a `CLAUDE.md` from `templates/CLAUDE.md`, filled in for our stack and constraints — but **do not overwrite an existing `CLAUDE.md`**; show me a diff to merge.
> 2. Add `PLAN.md` and `QA_CASES.md` from the templates.
> 3. Treat `workflows/engineering-principles.md` as standing instructions and link it from our `CLAUDE.md`.
> 4. Wire the gates in `workflows/` and `skills/` to our actual CI, error tracker, and PR-review bot.
> 5. Propose any **stack-specific** adjustments to the installed commands and skills, and create them under our agent-config once I approve.
>
> Explain what you changed and why. Commit nothing until I review.

Review the diff, then commit. The agent reads the doctrine and tailors *your* version — that's the point. It's better than copying someone else's skill files blind, because the result fits your stack instead of a stranger's.

## What you end up with

- **The command surface** — `/brainstorm`, `/plan`, `/peer-review`, `/release-prep`, `/release-prod`, `/checkpoint`, `/debug`, `/release-check`, `/principal-consultant`, and `engineering-principles`, installed and (optionally) tailored to your stack.
- **`CLAUDE.md`** — your project constitution (hard rules, always-current facts, conventions).
- **Plan-state convention** — the `PLAN.md` header + durable-planning rules so multi-session work doesn't drift.
- **`QA_CASES.md`** — the living, only-grows QA list.
- **A review + release gate** — cross-model review + a staged, human-authorized release path, wired to *your* tools.

## Prefer to do it by hand?

1. Install the command surface as above (symlink `workflows/` → commands, `skills/` → skills).
2. Copy `templates/CLAUDE.md`, `templates/PLAN.md`, `templates/QA_CASES.md` into your repo; fill every `<placeholder>`.
3. Make `workflows/engineering-principles.md` your agent's standing instructions (link it from `CLAUDE.md`).
4. Read `concepts/`, `workflows/`, and `skills/`, and wire the gates to your CI / error tracker / PR-review bot.

## Customize for your world

- **Stack** — swap the generic examples for your languages, CI runner, hosting target, and error tracker. The release workflows (`workflows/release-prep.md`, `workflows/release-prod.md`) are where most of that wiring lives.
- **Team vs. solo** — the gates apply either way; only *how* they map differs. The canonical breakdown lives in [`workflows/release-prep.md`](workflows/release-prep.md).

## Access

While this repo is private, point your agent at a **local clone** (or get collaborator access first) — an agent can't read a private repo from a URL without your credentials. Once it's public, the URL works directly.

## Keep it in sync

It's a living playbook. `git pull` for updates — if you symlinked the command surface, the live commands update in place; otherwise re-run the relevant install step. The principles change rarely; the tool-wiring in the release workflows is what you'll revisit as your stack evolves.
