---
name: debug
description: Systematic, root-cause-first debugging. Use when a test fails, a bug reproduces, or behavior is unexpected. Enforces "no fix before the root cause is found," one-variable hypothesis testing, and a 3-failures-means-architecture escalation rule. Also the remediate path for cross-model review findings.
---

# /debug — Systematic Debugging

*(adapted from [obra/superpowers](https://github.com/obra/superpowers) `systematic-debugging`, MIT — with thanks)*

Install per [SETUP.md](../../SETUP.md). This skill is the home for the systematic-debugging method — it states the doctrine and the procedure.

When something breaks — a failing test, a bug, unexpected behavior — the expensive mistake is guessing. Random fixes mask the cause and breed new bugs. Systematic is *faster* than guess-and-check; the thrash is the slow path.

## The iron law

> [!CAUTION]
> **No fix before the root cause is found.** A symptom fix is a failure, however tempting under time pressure. If you're about to change code before you can name *why* the value is wrong, stop.

## Four phases — finish each before the next

1. **Find the root cause.** Read the error and stack trace in full (the answer is often right there). Reproduce it reliably — if you can't, gather more data, don't guess. Check what recently changed (`git log`, `git diff`). In a multi-component system, instrument each boundary (what goes in, what comes out) and run *once* to find which layer fails before touching any of them. Trace the bad value back to where it originates; fix it there, not where it surfaces.
2. **Analyze the pattern.** Find similar code that works; list every difference, however small ("that can't matter" is where bugs hide). Understand the dependencies and assumptions.
3. **Hypothesize and test.** State one hypothesis ("X is the cause because Y"). Make the smallest change that tests it — one variable at a time. Worked → phase 4. Didn't → new hypothesis; don't stack fixes.
4. **Fix at the root.** Write the failing test first (add it to your `QA_CASES.md` — see [the template](../../templates/QA_CASES.md)), then the single fix, then verify it passes and nothing else broke. No "while I'm here" changes.

## The escalation rule

> [!IMPORTANT]
> **Three failed fixes = an architecture problem, not a fourth fix.** If each attempt reveals a new problem elsewhere, or fixes need "massive refactoring," stop and question the design with a human. That pattern means the approach is wrong, not the attempt.

## Red flags — stop, return to phase 1

"Quick fix now, investigate later" · "just try changing X" · "it's probably X, let me fix it" · proposing a fix before tracing the data · a fourth attempt after three failures.

## Self-improvement hook

Every root cause found here ends as a permanent failing-then-passing case in your `QA_CASES.md` so the bug can't silently return — this is the **bug-triggered half** of the self-improving QA loop. The session-ending half is checkpointing (see the lifecycle's [Checkpointing](../../concepts/ai-dev-lifecycle.md#checkpointing) section). Same goal: the system gets measurably better every cycle.

This skill is also the **remediate path** for findings from [Cross-Model Adversarial Review](../peer-review/SKILL.md): when a review pass cites a real defect, run it back through the four phases above rather than patching the symptom the reviewer pointed at.
