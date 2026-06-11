# The AI Development Lifecycle

How a unit of work moves from intent to shipped code with an AI agent doing the building. Governing idea: **plan the work, then work the plan.** Humans are bad at being clear about what they want; agents are aggressive about whatever goal they're given. Structure closes that gap.

Pairs with [Engineering Principles](../principles/ENGINEERING-PRINCIPLES.md) (what not to do), [Durable Planning](../planning/DURABLE-PLANNING.md) (how the plan survives across sessions), and [Debugging](DEBUGGING.md) (when something breaks mid-build). Several disciplines below are adapted from [obra/superpowers](https://github.com/obra/superpowers) (MIT), credited inline.

## Phase 1 — Intent → requirements (with a hard gate)

A human states the intent; the agent acts as a senior PM — asks clarifying questions *one at a time* (not a barrage), names users, proposes 2-3 approaches with a recommendation, flags risks.

**Hard gate** *(adapted from superpowers `brainstorming`)*: **no code until a design is approved** — for *every* task, regardless of apparent simplicity. "Too simple to need a design" is exactly where unexamined assumptions waste the most work. The design can be three sentences for a trivial change; it still gets written and approved.

A useful requirements structure is **four iterative rounds** *(round structure: Kris Skrinak's ContextEng)*: foundation (users / journeys / shape) → technical depth (data / deps / security) → implementation plan (environments / testing / monitoring) → task decomposition.

> A two-hour requirements pass prevents twenty hours of rework.

## Phase 2 — Requirements → task list

Decompose into an ordered, dependency-aware checklist of **atomic** tasks — each completes in one agent session, touches a bounded set of files, has a clear "done" test.

**Author tasks to be executable by a low-context engineer** *(adapted from superpowers `writing-plans`)*:
- **Bite-sized steps** (~2-5 min each): write the failing test → run it (watch it fail) → minimal code → run it (watch it pass) → commit.
- **No placeholders.** "TBD," "add error handling," "write tests for the above," "similar to Task N" are *plan failures*. Show the actual code, exact file paths, exact commands + expected output.
- **Self-review the plan:** every requirement maps to a task (coverage), no placeholders, type/name consistency across tasks.

This is the *single-plan executability* axis; the *cross-session durability* axis (plan-state headers, anti-drift) is [Durable Planning](../planning/DURABLE-PLANNING.md). Use both.

## Phase 3 — Execute (subagent-driven)

*(adapted from superpowers `subagent-driven-development`)* Dispatch a **fresh subagent per task** — isolated context keeps it focused and keeps your coordinating context clean. Construct exactly the context it needs; hand it the task's full text rather than making it read the whole plan.

Each task runs:
1. **Implement** against the spec — search existing code first, match patterns (Principles §10), write the test, then the code.
2. **Spec-compliance review** (a reviewer subagent): does it match the spec — nothing missing, nothing extra? Fix → re-review until clean.
3. **Code-quality review** (a second reviewer): *only after* spec passes. Fix → re-review until clean.
4. **Lint / typecheck, mark done, commit.**

Handle subagent status honestly: `BLOCKED` / `NEEDS_CONTEXT` means *something must change* — more context, a stronger model, a smaller task, or escalate — never a silent retry. **Tier the model:** cheap for mechanical tasks, capable for design and review. Don't pause for "should I continue?" between tasks — execute the plan.

*(This is the execution loop. The PR-level cross-model review + automated gate in [release-and-review](../release-and-review/CROSS-MODEL-REVIEW.md) is a separate, later layer.)*

## Context discipline — context is the scarce resource

Agent quality degrades two ways: **context rot** (critical detail buried as the window fills) and **catastrophic forgetting** (polluted context → contradictions). Every always-loaded token also competes with the task, and **duplication causes drift** — the same rule in three files becomes three rules that diverge.

- **Clear and reload** at ~80%, between phases, before hard problems — from the source of truth (constitution + plan + task state).
- **One canonical home per concept; reference, don't restate.** A copy-pasted paragraph is a bug.
- **Tier by load-frequency.** Keep "always-loaded" (the constitution) minimal; push "sometimes-needed" detail to `docs/` and load on demand.
- **Compose, don't inline.** A shared procedure lives in one referenced file — not a copy in every workflow.
- **Isolate.** Delegate fan-out (search, investigation) to subagents so their reads don't pollute the main context.

## Checkpointing

Treat state as lost at any moment. Commit after each task (a pushed commit is the only proof it's done); persist what's done / next / decided to durable files, not the window (Principles §12); close every session by updating the state doc + plan headers.

**This is half of a self-improvement loop.** Two triggers feed it: a **bug** → harden the tests so it can't recur ([Debugging](DEBUGGING.md) finds the root cause; a failing-test entry in `QA_CASES.md` locks it in); a **session ending** → capture what was learned — decisions, knowledge gaps, corrections — into durable state. Same goal (the system gets measurably better every cycle), two entry points. Give each kind of learning **one home** — a recurring pattern, a process rule, a fact, and a bug-test each have a single canonical destination — so the learnings don't drift across files.

## When the task list is done but work remains

Archive the finished list and generate a fresh one from requirements, or open a `next-steps` doc through the same Phase 1→2 process. Don't let a stale list accrue half-done items.
