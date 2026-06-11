# Systematic Debugging

*(adapted from [obra/superpowers](https://github.com/obra/superpowers) `systematic-debugging`, MIT — with thanks)*

When something breaks — a failing test, a bug, unexpected behavior — the expensive mistake is guessing. Random fixes mask the cause and breed new bugs. Systematic is *faster* than guess-and-check; the thrash is the slow path.

## The iron law

**No fix before the root cause is found.** A symptom fix is a failure, however tempting under time pressure.

## Four phases — finish each before the next

1. **Find the root cause.** Read the error and stack trace in full (the answer is often right there). Reproduce it reliably — if you can't, gather more data, don't guess. Check what recently changed. In a multi-component system, instrument each boundary (what goes in, what comes out) and run *once* to find which layer fails before touching any of them. Trace the bad value back to where it originates; fix it there, not where it surfaces.
2. **Analyze the pattern.** Find similar code that works; list every difference, however small ("that can't matter" is where bugs hide). Understand the dependencies and assumptions.
3. **Hypothesize and test.** State one hypothesis ("X is the cause because Y"). Make the smallest change that tests it — one variable at a time. Worked → phase 4. Didn't → new hypothesis; don't stack fixes.
4. **Fix at the root.** Write the failing test first, then the single fix, then verify it passes and nothing else broke. No "while I'm here" changes.

## The escalation rule

**Three failed fixes = an architecture problem, not a fourth fix.** If each attempt reveals a new problem elsewhere, or fixes need "massive refactoring," stop and question the design with a human. That pattern means the approach is wrong, not the attempt.

## Red flags — stop, return to phase 1

"Quick fix now, investigate later" · "just try changing X" · "it's probably X, let me fix it" · proposing a fix before tracing the data · a fourth attempt after three failures.
