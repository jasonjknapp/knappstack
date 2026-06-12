---
name: brainstorm
description: Hard design gate before any build. No code, no specs, no file changes until a design is approved — for every task regardless of apparent simplicity. Drives requirements as a senior PM (one question at a time, 2-3 approaches with a recommendation), structured in four iterative rounds.
---

# /brainstorm — Design Before Build (Hard Gate)

*(gate adapted from [obra/superpowers](https://github.com/obra/superpowers) `brainstorming`, MIT; four-round structure credit: Kris Skrinak's ContextEng — see [CREDITS](../../CREDITS.md). Install per [SETUP](../../SETUP.md).)*

Humans are bad at being clear about what they want; agents are aggressive about whatever goal they're given. This gate closes that gap before a line of code is written.

> [!CAUTION]
> **No code, no specs, no file changes until a design is approved — for *every* task, regardless of apparent simplicity.** "Too simple to need a design" is exactly where unexamined assumptions waste the most work. The design can be three sentences for a trivial change; it still gets written and approved first.

## Act as a senior PM, not a stenographer

- **Ask clarifying questions one at a time**, not as a barrage. Each answer shapes the next question.
- **Name the users and the journeys.** Who hits this, in what flow, to what end.
- **Propose 2-3 approaches with an explicit recommendation** and the trade-offs — don't just transcribe the human's first idea. Challenge it if it's suboptimal (the trusted-advisor stance applied to design).
- **Flag risks and unknowns** before they become rework.

## Four iterative rounds

1. **Foundation** — users, journeys, the shape of the thing.
2. **Technical depth** — data model, dependencies, security, failure modes.
3. **Implementation plan** — environments, testing strategy, monitoring.
4. **Task decomposition** — hand off to durable planning for the atomic task list.

> A two-hour requirements pass prevents twenty hours of rework.

## The gate is explicit

State plainly: **"Design approved?"** Do not proceed to planning or any build step until the human says yes. This is the design-side twin of a communication gate (approve the message before you send it) — same discipline (approve before you ship), one for code, one for communication.

## Where this sits in the lifecycle

This is Phase 1 of the [AI Development Lifecycle](../../concepts/ai-dev-lifecycle.md): intent → requirements, with a hard gate. The approved design flows into:

[Durable Planning](../plan/SKILL.md) (turn the approved design into a durable, atomic task list) → build → [Cross-Model Review](../peer-review/SKILL.md) → [Staged Release](../../workflows/release-prep.md).
