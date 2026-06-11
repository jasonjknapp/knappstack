# knappstack

*The agentic-engineering stack I bring to every build — an opinionated operating system for shipping production software with AI coding agents, not demos.*

> Principles, a development lifecycle, review + release discipline, and durable planning — the methodology I actually use, generalized for any team or solo builder. **New here? → [SETUP.md](SETUP.md).**

> **Status: v0.1** — extracted and generalized from a private working set, sanitized for public use. Issues and PRs welcome.

## Why this exists

Most "AI coding" advice stops at prompt tips. Shipping real software with agents is a different problem: the agent is *aggressive* about reaching its goal, context degrades over long sessions, and "it compiled" is not "it works." This repo is the operating model that closes that gap — the decisions you make **once, in advance**, so you're not re-litigating them every session.

The throughline: **AI drives, humans decide.** The agent proposes architecture, decomposes work, writes code, runs tests. You set intent, hold the guardrails, and own the result.

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

1. Drop the `CLAUDE.md` template into your repo and fill in your stack and constraints.
2. Adopt the principles as your agent's standing instructions.
3. Wire the release gate **before** you let agents touch anything shippable.

It's harness-agnostic in spirit (the principles hold for any capable coding agent) and was hardened in practice on Claude Code.

## Provenance & credits

This is distilled from hands-on delivery, and it stands on real shoulders — see [CREDITS.md](CREDITS.md). Short version: the context discipline builds on Anthropic's Claude Code work; the AWS-native agent architecture and several framing devices are adapted from [Kris Skrinak's](https://github.com/skrinak) open work; the rest is the broader agentic-coding canon plus what survived contact with production.

— Jason Knapp
