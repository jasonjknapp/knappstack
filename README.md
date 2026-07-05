# knappstack

*The agentic-engineering stack I use every day: An operating system for shipping production software with AI coding agents, not demos.*

> Principles, a development lifecycle, review + release discipline, and durable planning — the methodology I actually use, generalized for any team or solo builder.
>
> Based on principles gleaned from others' best practices (See [CREDITS.md](CREDITS.md)).
>
> **New here? → [SETUP.md](SETUP.md).**

> **Status: v0.1** — extracted and generalized from a private working set, sanitized for public use. Issues and PRs welcome.

## Who it's for

A **product/engineering team** standing up an AI-first delivery practice, or an **independent consultant** running that practice across clients. The principles and planning conventions apply to delivery work in general; the lifecycle and release sections are code-specific.

## What's inside — the command map

The doctrine essays describe the *shape* of each gate; the command files are the *runbooks*. Each command has one canonical home, and the concept essays reference those homes instead of restating them. Layout:

- **`workflows/`** — `/`-style command runbooks (the long-form executable workflows).
- **`skills/`** — model-invoked skills, one directory per skill with a `SKILL.md`.
- **`concepts/`** — the *why*/overview essays that tie the commands together.
- **`templates/`** — copy-ready `CLAUDE.md`, `PLAN.md`, and `QA_CASES.md` starters.

### Commands

| Command | Surface | One-line purpose |
|---|---|---|
| [`/brainstorm`](skills/brainstorm/SKILL.md) | skill | Hard design gate — no code until a design is approved; drives requirements as a senior PM. |
| [`/plan`](skills/plan/SKILL.md) | skill | Durable planning — writes a plan file with a PLAN-STATE header so decisions survive across sessions. |
| [`/principal-consultant`](skills/principal-consultant/SKILL.md) | skill | Communication + advisory protocol; the Launch-Email value gate, the prose analog of `/brainstorm`'s design gate. |
| [`engineering-principles`](workflows/engineering-principles.md) | workflow | The decision framework — consult before any task. |
| [`/release-check`](skills/release-check/SKILL.md) | skill | Fast local pre-PR sanity gate (build/lint/type/tests + secrets + git hygiene), under a minute. |
| [`/peer-review`](skills/peer-review/SKILL.md) | skill | Autonomous hardening — adversarial cross-model review, drives to convergence, opens a bot-clean PR. |
| [`/debug`](skills/debug/SKILL.md) | skill | Root-cause-first systematic debugging; also the remediate path for `/peer-review` findings. |
| [`/release-prep`](workflows/release-prep.md) | workflow | Pre-release pipeline — hardens, simplifies, clears the PR-review bot, verifies the build, waits for approval. Does **not** merge. |
| [`/release-prod`](workflows/release-prod.md) | workflow | Production release — runs after the staging gate; this is where the merge and distribution happen. |
| [`/checkpoint`](workflows/checkpoint.md) | workflow | Mid-session save — aligns docs with the session's work and writes a cross-conversation handoff. |

### The pipeline

The commands compose into one delivery flow:

```
/brainstorm  →  /plan  →  build  →  /peer-review  →  /release-prep  →  /release-prod
 (design gate)  (durable    (agent    (cross-model      (staging gate,    (merge +
                 plan)       builds)    hardening →       human-approved)   distribute)
                                        bot-clean PR)
```

Cross-cutting, available at any phase:

- **`/checkpoint`** — save state and a handoff whenever a session is about to end or context is filling up.
- **`/debug`** — drop in whenever something breaks mid-build; it's also where `/peer-review` findings get remediated.

`engineering-principles` and `/principal-consultant` are standing instructions — consulted *throughout*, not at a single step.

### Canonical homes (the *why* lives with each command)

| Topic | Path |
|---|---|
| Lifecycle overview | [`concepts/ai-dev-lifecycle.md`](concepts/ai-dev-lifecycle.md) |
| Principles | [`workflows/engineering-principles.md`](workflows/engineering-principles.md) |
| Cross-model review | [`skills/peer-review/SKILL.md`](skills/peer-review/SKILL.md) |
| Staged release | [`workflows/release-prep.md`](workflows/release-prep.md) → [`workflows/release-prod.md`](workflows/release-prod.md) |
| Durable planning | [`skills/plan/SKILL.md`](skills/plan/SKILL.md) |
| Systematic debugging | [`skills/debug/SKILL.md`](skills/debug/SKILL.md) |
| Self-improving QA loop | [`concepts/self-improving-qa-loop.md`](concepts/self-improving-qa-loop.md) |
| Local-first orchestration (cheap/local volume, frontier verify) | [`concepts/local-first-orchestration.md`](concepts/local-first-orchestration.md) |

## One canonical instruction file

Every agent CLI looks for its own instruction filename — Claude Code reads
`CLAUDE.md`, Gemini reads `GEMINI.md`, and `AGENTS.md` is the emerging cross-tool
standard (read natively by Claude Code, Codex, and others). Maintaining a separate
hand-edited copy per tool is how **instruction drift** starts: the copies fall out
of sync and agents follow stale rules.

The fix is one source of truth. Copy [`templates/AGENTS.md`](templates/AGENTS.md)
into your repository root as `AGENTS.md`, fill in the placeholders, then make each
tool's filename a **symlink** to it (run from the repo root):

```sh
cp path/to/knappstack/templates/AGENTS.md AGENTS.md   # then fill it in
ln -s AGENTS.md CLAUDE.md
ln -s AGENTS.md GEMINI.md
```

Now there is exactly one file to edit, and every harness reads the same rules. If
a tool ever needs a genuine delta, convert its symlink into a thin stub that
imports the canonical file — never a fork.

**Drift check (optional but recommended):** a tiny CI/pre-commit script can assert
that each instruction surface is still a symlink (or hash-match) to the canonical
`AGENTS.md`, and fail loudly if someone replaces one with a divergent copy. A
broken link becomes a build error instead of silent drift.

## How to use it

You can just point your AI at this repo and ask it to incorporate the principles into your own workflows, or you can take the whole thing and drop it in verbatim. If you choose the latter (designed for Claude):
1. Drop the `CLAUDE.md` template into your repo and fill in your stack and constraints.
2. Adopt the principles as your agent's standing instructions.
3. Wire the release gate **before** you let agents touch anything shippable.

It's harness-agnostic in spirit (the principles hold for any capable coding agent) and was hardened in practice on Claude Code.

## Provenance & credits

This is distilled from hands-on delivery but draws on others' best practices as well [CREDITS.md](CREDITS.md).
— Jason Knapp
