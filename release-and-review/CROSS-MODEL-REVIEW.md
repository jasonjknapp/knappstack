# Cross-Model Adversarial Review

Drive code from "in progress" to "review-clean" by having it reviewed adversarially by *different* models before it reaches a human — because a single model has consistent blind spots, and a second model (or an automated reviewer) catches a different class of problems.

This is the hardening stage that feeds [Staged Release](STAGED-RELEASE.md).

---

## Why different models

One model, reviewing its own work, misses what it's predisposed to miss. The value is in the *diversity of blind spots*:

- **Model A vs. Model B** (e.g. two different frontier models) — catches correctness, architecture, and reasoning gaps.
- **An automated PR-review bot** (e.g. a hosted AI code reviewer) — catches pedantic hygiene the cross-model pass glosses over.

These are complementary, not redundant. Run both; treat them as different lenses.

## The review rubric

Every review pass — by any reviewer — checks the diff against a fixed rubric, and **cites file + line + the failure scenario** for each finding:

1. **Correctness** — does it do what the task required?
2. **Goal alignment** — does it serve the stated objective, not a plausible adjacent one?
3. **Security** — inputs, authz, secrets, injection surfaces.
4. **Edge cases** — boundaries, empty/null, error paths, state transitions.
5. **Conventions** — the repo's established patterns and naming.
6. **Data safety** — anything that writes, deletes, or migrates.

## The anti-hallucination gate

Reviewers invent plausible-but-fake findings. Gate every finding:

- It must point to a **real file + line** and a **concrete, reproducible failure**.
- If the reviewer can't substantiate it, **default to "not a bug."**
- Style nits are not findings unless they violate a stated convention.

## The convergence loop

1. **First pass** — self/subagent review against the rubric. Fix the real 🔴/🟡 findings.
2. **Cross-model pass** — hand the diff to a *different* model with the same rubric. Fix what it surfaces.
3. **Repeat** until **two consecutive clean passes from different models.** Any fix voids convergence — re-run.
4. **Cap the rounds** (e.g. 4). If it won't converge, escalate to a human — non-convergence is itself a signal.

## The pattern bank

Recurring findings are cheaper to prevent than to re-review:

- Keep a **pattern bank** — a running list of issues reviewers have flagged before.
- Before each review, **grep the diff for every pattern** in the bank and fix matches preemptively.
- When a reviewer surfaces a new recurring class, add it to the bank.

## Then the automated gate

Only after cross-model convergence: push the branch, open the PR, and let the **automated PR-review gate** run to clean. Notes:

- Let it auto-review on push; don't manually re-trigger while a review is in flight (doubles volume for no signal).
- Teach it your conventions up front (a styleguide doc it honors) so it stops re-flagging accepted patterns.
- Cross-model convergence and the automated gate are **not the same check** — the former is correctness/architecture, the latter is hygiene. Both must pass.

---

**One line:** rubric-driven, anti-hallucination-gated, two-clean-passes-from-different-models, then the automated gate — and a pattern bank so the same finding never costs you twice.
