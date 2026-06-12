---
description: Why/overview for the AI development lifecycle — how a unit of work moves from intent to shipped code with an agent building it, and which command owns each phase.
---

# The AI Development Lifecycle

How a unit of work moves from intent to shipped code with an AI agent doing the building. Governing idea: **plan the work, then work the plan.** Humans are bad at being clear about what they want; agents are aggressive about whatever goal they're given. Structure closes that gap.

This is the **overview** — the *why*, and the map of which command owns each phase. Each phase has one canonical home that holds the method; this essay references those homes instead of restating them (context-DRY — a copy-pasted procedure is a future contradiction). Pairs with [Engineering Principles](../workflows/engineering-principles.md) (what not to do) and [Durable Planning](../skills/plan/SKILL.md) (how the plan survives across sessions). Several disciplines below are adapted from [obra/superpowers](https://github.com/obra/superpowers) (MIT), credited at their command homes.

## The throughline

**AI drives, humans decide.** The agent proposes architecture, decomposes work, writes code, runs tests. You set intent, hold the guardrails, own the result. The lifecycle is the set of decisions you make **once, in advance** — a hard design gate, a durable plan, an isolated execution loop, an adversarial review — so you're not re-litigating them every session.

## Phase 1 — Intent → requirements (the hard design gate)

A human states the intent; the agent acts as a senior PM — asks clarifying questions *one at a time* (not a barrage), names users, proposes 2-3 approaches with a recommendation, flags risks. A useful requirements structure is **four iterative rounds**: foundation (users / journeys / shape) → technical depth (data / deps / security) → implementation plan (environments / testing / monitoring) → task decomposition.

The non-negotiable: **no code until a design is approved** — for *every* task, regardless of apparent simplicity. "Too simple to need a design" is exactly where unexamined assumptions waste the most work. A two-hour requirements pass prevents twenty hours of rework. The design can be three sentences for a trivial change; it still gets written and approved.

**Home: `/brainstorm`** — the design gate. It owns the one-question-at-a-time interrogation, the 2-3-approaches-with-a-recommendation shape, and the four-round structure. Run it first.

## Phase 2 — Requirements → durable plan

Decompose into an ordered, dependency-aware checklist of **atomic** tasks — each completes in one agent session, touches a bounded set of files, has a clear "done" test. Author tasks to be executable by a low-context engineer: bite-sized steps (write the failing test → watch it fail → minimal code → watch it pass → commit), **no placeholders** ("TBD," "add error handling," "similar to Task N" are *plan failures* — show the actual code, exact paths, exact commands and expected output), and a self-review that every requirement maps to a task with no type/name drift across tasks.

That's the *single-plan executability* axis. The *cross-session durability* axis — plan-state headers, one routing layer, cite-don't-paraphrase, verbatim goal capture — is what keeps the plan from rotting across weeks of sessions.

**Home: `/plan`** ([Durable Planning](../skills/plan/SKILL.md)) — owns the PLAN-STATE v1 header, the single-routing-layer rule, and the anti-drift conventions. Both axes apply; use them together.

## Phase 3 — Execute (subagent-driven)

Dispatch a **fresh subagent per task** — isolated context keeps it focused and keeps your coordinating context clean. Construct exactly the context it needs; hand it the task's full text rather than making it read the whole plan. The shape is generic and harness-agnostic: a fresh CLI subagent, given a model and an effort/reasoning level appropriate to the task, that writes its result to an output file you then read back.

Each task runs:

1. **Implement** against the spec — search existing code first, match patterns (Principles §10), write the test, then the code.
2. **Spec-compliance review** (a reviewer subagent): does it match the spec — nothing missing, nothing extra? Fix → re-review until clean.
3. **Code-quality review** (a second reviewer): *only after* spec passes. Fix → re-review until clean.
4. **Lint / typecheck, mark done, commit.**

Handle subagent status honestly: `BLOCKED` / `NEEDS_CONTEXT` means *something must change* — more context, a stronger model, a smaller task, or escalate — never a silent retry. **Tier the model:** cheap for mechanical tasks, capable for design and review. Don't pause for "should I continue?" between tasks — execute the plan.

**Home: the subagent execution loop in `/peer-review` (Step -1.3a).** It owns the fresh-subagent dispatch shape, the spec-then-quality review ordering, and the honest-status handling. This is the *execution* loop; the PR-level cross-model review + automated gate is a separate, later layer — see [Cross-Model Review](../skills/peer-review/SKILL.md).

## When something breaks mid-build

A failing test or unexpected behavior interrupts execution. The expensive mistake is guessing — random fixes mask the cause and breed new bugs. The iron law: **no fix before the root cause is found.** Trace the bad value to where it originates and fix it there, test one variable at a time, and treat **three failed fixes as an architecture problem, not a fourth fix.**

**Home: `/debug`** — owns root-cause-first debugging and the escalation rule. It's also the remediate path for findings surfaced by `/peer-review`.

## Context discipline — context is the scarce resource

Agent quality degrades two ways: **context rot** (critical detail buried as the window fills) and **catastrophic forgetting** (polluted context → contradictions). Every always-loaded token competes with the task, and **duplication causes drift** — the same rule in three files becomes three rules that diverge. (This essay obeys that: each phase's method lives in one command, referenced here, not copied.)

- **Clear and reload** at ~80%, between phases, before hard problems — from the source of truth (constitution + plan + task state).
- **One canonical home per concept; reference, don't restate.** A copy-pasted paragraph is a bug.
- **Tier by load-frequency.** Keep "always-loaded" (the constitution) minimal; push "sometimes-needed" detail to on-demand docs.
- **Compose, don't inline.** A shared procedure lives in one referenced file — not a copy in every workflow.
- **Isolate.** Delegate fan-out (search, investigation) to subagents so their reads don't pollute the main context.

## Checkpointing

Treat state as lost at any moment. Commit after each task (a pushed commit is the only proof it's done); persist what's done / next / decided to durable files, not the window (Principles §12); close every session by updating the state doc + plan headers ([Durable Planning](../skills/plan/SKILL.md)).

**This is half of a self-improvement loop.** Two triggers feed it: a **bug** → harden the tests so it can't recur (`/debug` finds the root cause; a failing-test entry in `QA_CASES.md` locks it in); a **session ending** → capture what was learned — decisions, knowledge gaps, corrections — into durable state. Same goal (the system gets measurably better every cycle), two entry points. Give each kind of learning **one home** — a recurring pattern, a process rule, a fact, and a bug-test each have a single canonical destination — so the learnings don't drift across files.

## When the task list is done but work remains

Archive the finished list and generate a fresh one from requirements, or open a `next-steps` doc through the same Phase 1→2 process (`/brainstorm` → `/plan`). Don't let a stale list accrue half-done items.

---

**The map, one line each:** `/brainstorm` gates the design · `/plan` makes it durable · `/peer-review` Step -1.3a executes it in isolated subagents · `/debug` handles what breaks · [Cross-Model Review](../skills/peer-review/SKILL.md) → [Staged Release](../workflows/release-prep.md) take it to production. Install per [SETUP.md](../SETUP.md).
