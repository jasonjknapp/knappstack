# knappstack

*The agentic-engineering stack I bring to every build — an opinionated operating system for shipping production software with AI coding agents, not demos.*

> Principles, a development lifecycle, review + release discipline, and durable planning — the methodology I actually use, generalized for any team or solo builder. **New here? → [SETUP.md](SETUP.md).**

> **Status: v0.1** — extracted and generalized from a private working set, sanitized for public use. Issues and PRs welcome.

## Who it's for

A **product/engineering team** standing up an AI-first delivery practice, or an **independent consultant** running that practice across clients. The principles and planning conventions apply to delivery work in general; the lifecycle and release sections are code-specific.

## What's inside

| Area | Path | What it gives you |
|---|---|---|
| **Principles** | [`principles/ENGINEERING-PRINCIPLES.md`](principles/ENGINEERING-PRINCIPLES.md) | The decision framework — consult before any task. |
| **Lifecycle** | [`lifecycle/AI-DEV-LIFECYCLE.md`](lifecycle/AI-DEV-LIFECYCLE.md) | Intent → requirements (hard gate) → tasks → subagent-driven execution + context discipline. |
| **Debugging** | [`lifecycle/DEBUGGING.md`](lifecycle/DEBUGGING.md) | Root-cause-first systematic debugging for when something breaks mid-build. |
| **Review** | [`release-and-review/CROSS-MODEL-REVIEW.md`](release-and-review/CROSS-MODEL-REVIEW.md) | Adversarial review by different models, with an anti-hallucination gate. |
| **Release** | [`release-and-review/STAGED-RELEASE.md`](release-and-review/STAGED-RELEASE.md) | A staged, human-authorized path from clean code to production. |
| **Durable planning** | [`planning/DURABLE-PLANNING.md`](planning/DURABLE-PLANNING.md) | Plan-state headers + anti-drift conventions for multi-session work. |
| **Templates** | [`templates/`](templates/) | Copy-ready `CLAUDE.md`, `PLAN.md`, and `QA_CASES.md` starters. |
| **Setup** | [`SETUP.md`](SETUP.md) | How a team or solo dev adopts this in their own project (point your agent at it). |

## How to use it

You can just point you AI at this repo and ask it to incorporate the principals into your own workflows, or you can take the whole thing and drop it in verbatim. If you choose the later (designed for Claude):
1. Drop the `CLAUDE.md` template into your repo and fill in your stack and constraints.
2. Adopt the principles as your agent's standing instructions.
3. Wire the release gate **before** you let agents touch anything shippable.

It's harness-agnostic in spirit (the principles hold for any capable coding agent) and was hardened in practice on Claude Code.

## Provenance & credits

This is distilled from hands-on delivery but draws on others' best practices as well [CREDITS.md](CREDITS.md).
— Jason Knapp
