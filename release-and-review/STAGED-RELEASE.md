# Staged, Gated Release

The path from clean code to production. Every stage has a gate; nothing advances without passing it; **every production step is individually authorized by a human.** This is the discipline that makes Principles §5 (never deploy or mutate without permission) operational.

Pairs with [Cross-Model Adversarial Review](CROSS-MODEL-REVIEW.md), which is Stage 1's hardening engine.

---

## Stage 0 — Local pre-flight (fast, < 1 minute)

Before any review effort, prove the branch isn't trivially broken:

- Build, lint, typecheck, run the fast test suite.
- Git hygiene: right branch, clean tree, sane diff scope.
- **Secrets scan** — no keys/tokens/credentials in the diff.

Don't spend a full review round on a branch that fails this.

## Stage 1 — Harden + open the PR

- Run [Cross-Model Adversarial Review](CROSS-MODEL-REVIEW.md) to convergence (two clean passes from different models).
- Push the feature branch, open the PR, pass the **automated review gate** until clean.
- Exit criterion: a review-clean PR.

## Stage 2 — Staging

- **Identity check** — confirm you're on the right account and credentials *before* touching anything (a wrong-account deploy is a category of disaster).
- Verify code quality + any compliance requirements.
- Deploy to **staging**, not production.
- **Human QA** against the staging environment — test the business outcome (Principles §1/§6), not just that it loads.
- Exit criterion: **explicit human approval to promote.** No self-promotion.

## Stage 3 — Production

Each step below is a state change and gets its own human authorization:

1. **Identity lock** — re-confirm the production account/credentials.
2. **Back up first** — version control + a data backup (snapshot/export) *before* anything touches prod.
3. **Merge** to the release branch.
4. **Deploy** via the platform-appropriate path.
5. **Immediate smoke verification** — the critical paths work on prod.
6. **Soak** — watch your error tracker for a fixed window (e.g. 15 minutes) before declaring success.
7. **Canary verification** — confirm real traffic/usage looks healthy.
8. **Cautious resume** — bring any paused jobs/queues back deliberately, watching for the first signs of trouble.

If any step looks wrong, stop and roll back — a clean rollback beats a hopeful fix-forward under pressure.

---

## Tooling is yours to fill in

This describes the *shape* of the gate, not specific products. Slot in your own: CI runner, hosting/deploy target, error tracker (e.g. Sentry), automated PR reviewer, and backup mechanism. The discipline — gate every stage, authorize every production step, back up before you touch prod — is what transfers.

## For a solo operator vs. a team

- **Team:** the gates map to CI checks, required PR review, and a release owner who authorizes promotion.
- **Solo / consultant:** you *are* the release owner — the gates still apply, and the "explicit authorization" step is you deliberately deciding, not reflexively shipping. The cross-model review substitutes for the second pair of human eyes you don't have.
