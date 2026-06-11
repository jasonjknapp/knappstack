# Engineering Principles

A universal decision-making framework for building software with AI coding agents. Consult it before starting any task. The rules are deliberately terse and imperative — they're written to be loaded as an agent's standing instructions, not read as an essay.

Principles §1–§10 are general engineering discipline that AI agents make *more* necessary, not less. §11–§14 are agent-specific. Sources are credited in [CREDITS.md](../CREDITS.md).

---

## §1. Know Why You're Building

Every session begins with a clear objective. This is not optional.

1. **State the business objective before writing code.** "What outcome does this serve?" If you can't answer, stop and clarify with the human.
2. **Map it to a priority.** Every task should serve an active priority; work that serves none gets parked, not executed.
3. **QA against the objective, not the diff.** "Does this solve the user's problem?" — not just "does it compile?"
4. **Scope to the objective.** No speculative features, no premature abstractions, no "we might need this later." Build exactly what the objective requires, production-grade.

## §2. Prioritization Discipline

Agents are happy to do *something*; your job is to make sure it's the *right* thing.

1. **Hold a priority order and enforce it:** compliance/legal → committed deliverables (deadlines, promises) → quick wins (≤30 min, any tier) → cost/automation → revenue → everything else (backlog only).
2. **Pull quick wins up** from anywhere — zero easy items should age in a backlog.
3. **Challenge off-priority work.** If the current task serves no active priority, say so and propose parking or swapping it.

## §3. Pursue the Optimal Solution — No Workarounds

When you hit a missing field, missing access, or an architectural gap:

- **Pursue the correct fix first.** Weigh cost-per-request, latency, and architectural correctness.
- **Never add a runtime workaround** (e.g. an extra read on every request) when the right fix is to enrich data at the source (a one-time backfill, an added index field, a sync-pipeline change).
- A one-time backfill is far cheaper than N extra reads on every call, forever.

## §4. Ask for Access — Don't Work Around Permissions

If you need a key, credential, admin grant, or resource to do it properly:

- **Ask for exactly what you need, and why** — don't fall back to a workaround that avoids the access.
- Be specific: "I need X to run Y; can you provide it or point me to it?"

## §5. Production Safety — Never Deploy or Mutate Without Permission

- **Never push to a production service** (hosting, cloud, app stores) without explicit permission.
- **Never write/update/delete production data** without permission — including via scripts.
- **Present the plan first:** what you'll do, which systems/collections are touched, then wait for approval.
- **Dry-run first.** For any migration or backfill, run `--dry-run` and show results before going live.

## §6. Adversarial Testing — QA Is Not the Happy Path

- **The happy path is not QA.** Actively test error paths, boundaries, state transitions, and integration seams on the *first* pass.
- **Test against the objective** (§1), not just compilation.
- **Maintain a living QA case list** (e.g. `docs/QA_CASES.md`); run all relevant cases before calling work done.
- **When a bug is found, fix it, then re-run all relevant cases** — not just the one that failed.
- **The case list only grows.** Every new failure mode becomes a permanent case.
- **If the human has to ask "did you check edge cases?" — you failed.** Pre-empt it: list what you tested.

## §7. Anti-Deferral — Build It Right the First Time

- **Never propose "quick version now, proper version later"** unless the proper version is blocked by an external dependency that can't be resolved this session.
- **"It would take longer" is not a valid deferral reason.** The only valid ones: (1) missing access, (2) waiting on a third party, (3) a dependency that isn't ready, (4) explicit human override.
- **If you're tempted by a shortcut, state both options and let the human choose.** You don't get to unilaterally pick the band-aid.
- **Error handling, edge cases, validation, and tests ship *with* the feature** — not "added later."

## §8. Data at the Source

- Keep frequently-queried data close to where it's queried. If a search index or cache is in the hot path, make sure it carries every field its consumers need.
- Configure sync pipelines and extensions to include all downstream-required fields.
- When you add a field to a model, check whether indexes/caches need it too.

## §9. Blast-Radius Scoping

- **Fix everything inside the blast radius of your change.** Touching a function and you spot a bug next to it? Fix it now.
- **Issues *outside* the blast radius:** log them immediately (e.g. `docs/BACKLOG.md`) with severity and context. Don't silently ignore them; don't let them derail the task.
- **Definition of done:** production-grade on the first pass, scoped to the stated requirement.

## §10. Pattern Reuse — Never Build From Memory

When creating a file that shares structure with existing ones (components, configs, schemas, pages):

1. **Find the exemplar** — the most similar existing file.
2. **Copy, don't recreate.** Duplicate it and change only what differs.
3. **Never type from memory** values that already exist in the codebase (library URLs, font names, config keys, import paths). Copy the exact string from source — models hallucinate plausible-but-wrong values.
4. **Cross-reference after creating.** Diff the new file's shared parts against the exemplar; any unexplained difference is a bug.
5. **Stdlib over hand-rolled.** Prefer `JSON.stringify()` / `encodeURIComponent()` over manual escaping. If a standard function exists, use it.

---

## §11. Treat Permissions as Architecture

*(concept credit: Kris Skrinak's Simple-AI-DLC)*

- **Classify every tool the agent can reach by how much it can break** — reads that change nothing, writes that change state, and actions you can't take back.
- **Anything irreversible stops and waits for a human "yes"** — enforced in the harness (hooks, allowlists), not left to the agent's judgment.
- **A permission decision is a logged event you can review later**, not a prompt that flashes by and disappears.

## §12. Agents Crash — Plan for It

*(concept credit: Kris Skrinak's Simple-AI-DLC)*

- **Checkpoint continuously.** What you need to resume isn't the transcript — it's where the work stands, what's already taken effect, and what budget remains.
- **Separate "what we discussed" from "where the work is."** The first is a conversation log; the second answers "which step, what side effects already happened, is re-running this safe?"
- **The recoverable copy of that state lives in files / version control, never only in the agent's working memory** — so a fresh session on any machine picks up cleanly.

## §13. Verify the Work — and the Tooling That Produced It

*(concept credit: Kris Skrinak's Simple-AI-DLC)*

Two checks, both required:

1. **Is the task itself correct?** (That's §6.)
2. **Did changing your own tooling — a new command, hook, setting, or automation — quietly disable a guardrail?** After you touch the harness, re-prove the guardrails still fire. Tooling you haven't re-verified corrupts everything built on top of it, silently.

## §14. Rules Decay — Anchor Them to Intent, Not Specifics

*(concept credit: Kris Skrinak's ContextEng)*

The rule that bites you is the one that was right when written and quietly forbids the right answer later.

- **A hard "no exceptions" welded to a concrete implementation detail goes stale fast** — a blanket ban on some specific construct that, once the architecture moves, outlaws the very design you now need.
- **State the rule as the *reason* behind it, not the mechanism** — the outcome you're protecting, not which construct happens to be forbidden today.
- **When the ground shifts, rewrite the rule to its intent up front** — before it blocks live work, not after someone hits the wall.
- **Re-read your standing rules on a cadence:** sort the durable principles from the expired specifics, and rewrite or retire the latter.
