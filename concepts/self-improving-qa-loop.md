# The Self-Improving QA Loop

A test suite should get stronger with every bug it catches. The loop below is
how you make that automatic: **no failure mode goes uncaught twice.**

## The loop

After EVERY bug, incident, or unexpected behavior — found in review, in a release
gate, or in production:

1. **Fix it** (hotfix if critical; backlog it if not).
2. **Add a test** to the repo's QA case file (`docs/QA_CASES.md`) covering the
   exact failure mode — a corrections-as-tests entry, so the suite can never
   regress on this mistake again.
3. **Widen if cross-cutting.** If the bug could hit other repos/modules, add it
   to the shared regression pass.
4. **Capture the pattern.** If it reveals a recurring code or review pattern, add
   it to your review-patterns file (`docs/.ca-patterns.md`).
5. **Fix the process.** If it reveals a process failure, update the relevant
   command/workflow doc — not just the code.
6. **Post-mortem** any incident with user-facing impact.

## One destination per kind of learning (so copies don't drift)

The seam this closes: a single learning could plausibly go to several places, and
without a rule it lands inconsistently — then the copies diverge. Route each kind
to exactly ONE home:

| Kind of learning | Its one home |
|---|---|
| Recurring code / review pattern | your **known-patterns file** (e.g. `docs/review-patterns.md`) — the same one `release-prep` Phase 9 audits |
| Process rule / banned action | the relevant command or workflow doc |
| Durable fact / runbook | your project's knowledge / reference directory |
| A failing test for a specific bug | `docs/QA_CASES.md` |

This is the same anti-drift discipline that makes one canonical instruction file
(`AGENTS.md`) worth enforcing: knowledge, like instructions, should have exactly
one source of truth.

> Adapt the filenames to your stack; keep the invariant: **one destination per
> kind of knowledge, and every bug becomes a test.**
