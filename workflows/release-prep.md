---
description: End-to-end pre-release pipeline. Completes work, hardens code, converges cross-model, simplifies, opens the PR, clears the PR-review bot, verifies the release build, and waits for approval. Does NOT merge or distribute — that is the production-release workflow's job. Single entry point — invoke and walk away.
---

# Release Prep Workflow (`/release-prep`)

**Purpose:** Drive a changeset from "in progress" to "human-approved, staging-verified, merge-ready" in a single invocation. Orchestrates the cross-model hardening pass, the local pre-flight gate, and the simplification pass; inlines compliance, schema, accessibility, secrets, and staging steps.

**Entry criteria:** A feature branch with work on it. The branch may be incomplete, not-yet-reviewed, or partially hardened.

**Exit state:** A bot-clean, build-verified PR — **NOT merged, NOT distributed** — with the human's explicit go-ahead. The separate production-release workflow performs the merge → build → distribution.

> [!IMPORTANT]
> **`/release-prep` never distributes.** It hardens, opens the PR, clears the PR-review bot, and verifies the build compiles for release — then STOPS at an approved-but-unmerged PR. It does **NOT** merge to the trunk, does **NOT** trigger a CI distribution workflow, does **NOT** push to a release channel. The merge and the production push are the production-release workflow's responsibility. (Deploying to a true pre-prod **staging** environment — not production — still happens here in Phase 18; that is QA infrastructure, not a release. The production push is what moves to the production workflow.)

**Next step after approval:** Run the production-release workflow to merge and ship.

> **Do not rush.** Every shortcut here becomes a production incident later.

This is the orchestrator for the staging stages (work-completion → hardening → staging verification) and the [Cross-Model Adversarial Review](../skills/peer-review/SKILL.md) loop. Install per [SETUP.md](../SETUP.md).

---

## Design Principle — Minimize Bot Rounds

Every check that can run locally runs locally, BEFORE the first push. Compliance, schema validation, accessibility, the multi-pass review, simplification, and cross-model convergence all complete pre-push. Your PR-review bot (e.g. Gemini Code Assist, GitHub Copilot review, or similar) sees hardened code on round 1.

**Why it matters:** If compliance/schema/a11y/multi-pass checks run AFTER the PR is already open, any fix from those checks triggers another bot auto-review round. Target with this design: **1 bot round** per release. Your bot has a finite daily review quota; chasing extra rounds burns it for no new signal.

> [!IMPORTANT]
> **Walk-away mode.** This workflow is designed to run autonomously between natural human-interaction points. Expected stops:
> 1. Stale/dropped PR disposition (only if found in scope audit)
> 2. Large-scope fixes (>20 lines) during cross-model convergence — the sub-skill may ASK
> 3. A manual CI distribution trigger (platform-dependent)
> 4. Final approval at staging exit gate
>
> If none apply, this runs end-to-end without intervention.

---

## Stage A: Local Hardening (Pre-Push)

Every phase in this stage runs BEFORE the first `git push`. The goal is that the PR-review bot, when it auto-reviews the PR-opening push, has nothing substantive to find.

### Phase 0: Context & Identity Lock

1. **Identify the repo and workspace:**
   ```bash
   pwd
   git remote -v
   git config user.email
   git branch --show-current
   ```

2. **Cross-reference against your project/account registry:**
   - Verify the `git remote -v` org matches the expected project category
   - Verify `git config user.email` matches the registry entry for this repo
   - Verify you are on a **feature branch**, NOT the trunk or release branch
   - Set `PROJECT_CATEGORY` (e.g. `internal` | `personal` | `client`)

3. **Read-only repo HARD BLOCK:**
   - If the remote org is one you do not have release rights to, STOP: "This is a read-only repo. Cannot release."
   - Check your account registry for any per-repo contributing overrides.

4. **Working directory validation:**
   - Run `git diff --name-only main...HEAD` to see what files changed
   - Verify all changed files belong to THIS repo (not a parent or sibling repo)
   - If changes span multiple repos, STOP: "This changeset touches files outside this repo. Split into separate releases."

5. **Load repo context:**
   - View the repo's project constitution (`CLAUDE.md` and your bot's sacred-rules file) for repo-specific rules, deploy targets, and gotchas
   - View the repo's backlog (`docs/BACKLOG.md`) to understand what's in-flight
   - View the project's strategic-focus doc, if any — verify the work aligns with current priorities

6. **Detect platform type** (determines staging and deploy mechanics in Stage C):
   - native mobile app → your mobile CI
   - web app → your web host's preview/deploy flow
   - static site / hosted page → your hosting deploy
   - backend service → your function/service deploy (no hosting)

---

### Phase 1: Scope Audit & Releases-In-Flight

7. **Open PR & Scope Audit (Scope Lock):**
   - Run `gh pr list --state open`
   - For EACH open PR, determine its status by comparing timestamps:
     ```bash
     gh pr view <NUMBER> --json headRefName,commits --jq '.headRefName'
     git log --oneline --since="<PR creation date>" main
     ```
   - **Classify each open PR (multi-workspace aware):**
     - **⚪ Ignore (Active WIP):** Recently committed (within 12–24h) OR tracked in another window's `releases-in-flight.md` → IGNORE. Assumed active WIP elsewhere.
     - **🟢 Superseded:** Commits merged to the trunk after the PR's last touch on the same files → offer to close: "PR #XXX appears superseded by [commit]. Close it?"
     - **🟡 Dropped Work / Incomplete Release:** Unmerged, NOT recently active (>24h), belongs to this repo → STOP. "I found a past incomplete release PR #XXX. Rebase, merge, or close?"
     - **🔴 Dangerous:** Would REVERT newer code in the trunk → flag: "PR #XXX predates [newer commit] and would revert. Recommend closing."
   - Do NOT proceed until scope is locked and all stale/dropped PRs are dispositioned (ignoring active WIP).

8. **Clean up scratch files** left by previous AI sessions:
   ```bash
   git status --short | grep -E '^\?\? '
   ```
   If temporary `.py` / `.js` / `.sh` / `test_*.js` files exist, delete before proceeding.

9. **Create/update the releases-in-flight entry (MANDATORY):**
   A global file (`releases-in-flight.md` in your agent-config directory) tracks in-flight releases across all repos and windows. The production-release workflow's first phase requires an entry with status `approved` for this release.
   ```bash
   REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
   BRANCH=$(git branch --show-current)
   PR_NUM=$(gh pr view --json number -q '.number' 2>/dev/null || echo "pending")
   DATE=$(date +%Y-%m-%d)
   ```
   If the file doesn't exist, create it with `## In Flight` and `## Completed` sections. Append (or update) under `## In Flight`:
   ```markdown
   ### $REPO — $BRANCH
   - **PR:** #$PR_NUM
   - **Status:** `hardening` (→ `pr-opened` in Phase 14, → `staging-deployed` / `build-verified` in Phase 18, → `approved` at the exit gate. The merge + production push happen later in the production-release workflow, not here.)
   - **Opened:** $DATE
   - **Workspace:** $(pwd)
   ```

---

### Phase 2: Bot Debt + Complete Unfinished Work

**Follow** the cross-model hardening skill's **Bot-Debt Resolution** and **Complete Unfinished Work** phases.

- The bot-debt scan checks open PRs for unresolved PR-review-bot findings; if any exist, fix them, then **restart** from Phase 0 of THIS workflow.
- The unfinished-work check reads `handoff.md` and the implementation plan for incomplete items; executes them before any review work.
- It also bootstraps your known-patterns file and your bot's styleguide if missing.

Do not skip the restart rule. If any bot debt was fixed, the entire pipeline must re-run — those fixes are code changes that need the full hardening loop.

---

### Phase 3: Safety Property Enumeration

**Follow** the cross-model hardening skill's **Safety Property Enumeration** phase.

Enumerate safety-critical properties (no infinite loops, no data loss, no cost explosion, no schema drift, no auth bypass). Verify each with `grep` / schema checks / code-path tracing — not "it looks right."

---

### Phase 4: Build / Lint / Tests Gate

**Invoke the local pre-flight gate.** This runs:
- Build (`npm run build`, native build, `tsc`, etc.)
- Lint (`npm run lint`, language-specific analyzer, etc.)
- Type check (`tsc --noEmit`, language analyzer)
- Test suite (`npm test`, framework test runner)
- Docs sync check (`handoff.md`, `docs/BACKLOG.md`, project constitution, `docs/QA_CASES.md`)
- Git hygiene + deploy safety

**Gate condition:** Must return PASS. If FAIL, fix the failures and re-invoke. Do NOT advance to Phase 5 until the gate is clean.

**Strict mode:** Run linters at maximum strictness — treat info-level findings as failures, to catch what your PR-review bot would catch.

---

### Phase 5: Secrets Audit

> **Hard gate, not a suggestion.** Leaked tokens require key rotation.

10. **Scan for secrets in the diff:**
    ```bash
    git diff main...HEAD -- . ':!*.lock' ':!*.min.js' | grep -iE '(sk_live_|sk_test_|AIzaSy|AKIA[A-Z0-9]|ghp_|gho_|glpat-|xox[bps]-|Bearer [a-zA-Z0-9]{20,}|-----BEGIN (RSA |EC )?PRIVATE KEY)'
    ```
    Any match = 🔴 **BLOCKER**. Do not proceed. Remove the secret and rotate it.

11. **Verify .env files are gitignored:**
    ```bash
    git ls-files --cached | grep -E '\.env(\.local|\.staging|\.production)?$'
    ```
    Any tracked `.env` file = 🔴 **BLOCKER**.

12. **Verify required env vars exist for the target environment:**
    Check the repo's `.env.example` or docs for required variables. Verify they're set in the target environment (your host, your CI).

---

### Phase 6: Cross-Repo Schema Validation

> Skip this phase for projects with no cross-service data contracts.

13. **Detect schema-touching changes:**
    Grep the diff for:
    - Datastore collection/field names in read/write paths
    - Validators in your data-access layer
    - Shared model/type changes consumed across services
    - Query fields in any search-index or client-facing query code
    - Service response shapes

14. **If schema changes detected**, load your schema-dependency graph and **delegate to a fresh CLI subagent (high-capability model, high effort)** to analyze downstream impact:
    - Prompt it to analyze `git diff main...HEAD` for changes to schema, validators, models, or query fields, cross-reference against the dependency graph to find downstream impacts, and highlight breaking changes as 🔴 blockers.
    - Read the subagent's verdict from its output file.
    - 🔴 blocker if it flags any consumer breaks → fix before proceeding.
    - Flag any multi-version client regression test as mandatory for Phase 18.

---

### Phase 7: Compliance & Legal Gates

> Skip this phase for projects with no compliance/legal surface.

15. **Compliance Guardrails & Privacy Checks** — delegate to a fresh CLI subagent (high-capability model, high effort):
    - Prompt it to review `git diff main...HEAD` against your compliance guardrails and privacy/policy runbook to verify no legal/compliance boundary breaches, cite specific lines if violations are found, and flag any store/distribution declaration mismatches.
    - Any violation = 🔴 blocker → fix before proceeding.

---

### Phase 8: Accessibility Check

16. **Accessibility Scan** — delegate to a fresh CLI subagent (high-capability model, high effort):
    - Prompt it to scan `git diff main...HEAD` for accessibility violations (missing alts, ARIA labels, heading hierarchy, tap targets, semantics wrappers for native UI).
    - Convert reported findings to 🟡 should-fix or 🔴 blockers. Fix blockers before proceeding.

---

### Phase 9: Styleguide & Pattern-Bank Audit

17. **Load styleguide + pattern bank:**
    - View your bot's styleguide if present (conventions the bot already honors — violations will definitely be flagged)
    - View your known-patterns file if present (recurring bot findings with canonical fixes)
    - If either is missing, Phase 2 should have bootstrapped them. If not, create minimal versions now.

18. **Cross-file consistency + pattern check** — delegate to a fresh CLI subagent (high-capability model, high effort):
    - Prompt it to review `git diff main...HEAD` against the styleguide (if present), your known-patterns file, stdlib-preference rules, asset-size gates, and cross-file markup/meta-tag consistency constraints; report deviations in detail.
    - Fix any 🔴 blockers before proceeding.

19. **Apply the pre-push hardening sweeps** per the cross-model hardening skill's pre-push rules:
    1. Fan-out the fix (grep the pattern across the repo)
    2. Defensive-parsing sweep (risky casts → safer patterns)
    3. Wrapper-property audit
    4. Strict analyzer pre-push
    5. Pattern-bank check

---

### Phase 10: Code Simplification

**Invoke the simplification pass** (your code-simplifier skill/agent).

It runs against recently modified code and applies refinements that **preserve functionality** while improving clarity, consistency, and maintainability (project standards, reduced nesting, consolidated logic, no nested ternaries).

**Review each simplification before accepting.** After the simplifier's changes:
- Run the full Phase 4 (local pre-flight) gate again — build, lint, type check, tests must still pass.
- If anything regressed, revert the simplifier's change and proceed without it.

**Why here:** Simplification must run BEFORE cross-model convergence and BEFORE the first push. Simplification after the PR is open triggers another bot round. This placement lands simplification in the PR-opening bundle.

---

### Phase 11: Multi-Pass Code Quality Review

> Run for changes >50 lines or touching critical paths. For smaller changes, run the core passes (code quality, bug hunting, security, build hygiene) at minimum.

20. **Pre-Check: Business Objective Verification**
    - State the specific outcome this change delivers
    - Verify the diff actually delivers it — if a gap exists, stop and reconcile

21. **Multi-Pass Review** — delegate to a fresh CLI subagent (high-capability model, high effort), writing to a known output file:
    - Prompt it to perform a rigorous multi-pass review (UX, code quality, bug hunting, security, performance, data integrity, testing, build hygiene, cross-file consistency, SEO) on `git diff main...HEAD`, producing a severity-classified markdown report (🔴 blocker, 🟡 should fix, 🟢 minor).
    - Read the report from the output file.
    - Fix 🔴 blockers and 🟡 should-fix items in the working tree (do NOT commit separately — they land in the PR-opening bundle).
    - If fixes touch behavior, re-run Phase 4 (local pre-flight) to confirm nothing broke.

22. **Post-Check: Outcome Validation**
    - Re-state the business objective, confirm delivery
    - Present a findings summary: 🔴 → 🟡 → 🟢

---

### Phase 12: Cross-Model Convergence

**Follow** the cross-model hardening skill's **Convergence Loop** ([Cross-Model Adversarial Review](../skills/peer-review/SKILL.md)).

Run the CLI-subagent ↔ this-model adversarial review cycle until **two consecutive clean passes from different models.**

- Any fix applied during this phase invalidates the clean counter. Keep iterating until two consecutive cleans.
- Max 4 rounds before escalating to the human.
- For high-risk PRs (prod data mutation, data-access-layer changes, auth/billing/compliance), consider an optional third-model pass.

---

### Phase 13: Bot Simulation (Local Pre-Push)

**Follow** the cross-model hardening skill's **Bot Simulation Pass**.

Mechanically check each pattern in your known-patterns file and your bot's styleguide against the diff, then run a pedantic bot-style review via a CLI subagent (to a known output file).

**If any fixes are made:** convergence is void. Re-run Phase 12 before proceeding.

**Only advance to Stage B when bot simulation returns zero findings.**

---

## Stage B: Push, PR & Bot Gate

All local hardening is done. Stage B opens the PR and clears the real PR-review-bot gate. Goal: **clean bot review on round 1.**

### Phase 14: Push + PR + Bot Auto-Review

**Follow** the cross-model hardening skill's **Push, PR & Bot Gate** phase in full, including:

- Pre-push verification (identity, feature-branch, remote)
- Final pre-push pattern scan (known-patterns file + bot styleguide)
- Push and create the PR as **ready-for-review (not draft)** — most review bots skip drafts
- Bot auto-review wait (poll periodically, SHA-pinned), per-finding severity triage, fix-and-bundle discipline
- History audit + SHA-verified exit gate

**Severity-triage on bot return:**
- **Critical / High** → fix all, RETURN to Phase 12 (cross-model convergence re-runs on new HEAD), then another bot round.
- **Medium / Low / Info only** → default-backlog + reply with rationale. Low/Info → backlog or accept. **DO NOT run another bot round.** Proceed directly to Stage C.
- **Zero findings** → proceed directly to Stage C.

> [!CAUTION]
> **STOP-AT-MEDIUMS DISCIPLINE:** Once the bot returns zero open Critical/High on HEAD, **stop pushing reflexively.** Do not fix Mediums *unless you affirmatively decide they warrant a fix* and push another commit hoping to converge to "no findings at all." The bot will keep finding new Mediums each round (style, type-polish, defensive checks) — chasing that loop burns daily quota (review bots have a hard cap of a few reviews per PR per day) with zero new critical catches.
>
> **The rule is an escape hatch from endless cycles, not a gag on judgment.** You are explicitly authorized to fix Mediums when you affirmatively believe they should be fixed. Default to backlog when the finding is cosmetic / preference / style. Fix inline when the Medium identifies a genuine correctness issue your judgment says should not ship.
>
> **Default action on a Medium: backlog + reply with rationale.** Fix inline when EITHER condition holds:
> - The finding is a **user-facing correctness bug or deploy-blocker risk** (not "type could be tighter", not "console.log → console.warn", not "could be more defensive in case of a future change").
> - **Your judgment says it should be fixed even if it does not strictly meet the criteria above.** The criteria are a *floor* (always fix if they hold) — they do not bar you from fixing other Mediums when you affirmatively decide they warrant it.
>
> When you choose to fix:
> - The fix should be **genuinely small** — usually a single ≤5-line edit in a single file, no new abstractions. Accompanying test updates are fine. **Test updates are not a reason to backlog a genuine fix.**
> - The fix should be **guaranteed-safe** — doesn't change retry/error/schema semantics, doesn't touch live-pipeline code paths in ways that could introduce a regression.
> - You will trigger another bot round. Budget for it. Don't push a Medium-fix in the loop pattern (where you fix one, the bot finds another, you fix that…) — that's still off-limits and what STOP-AT-MEDIUMS exists to prevent.
>
> **What this rule kills**, which is still off-limits:
> - "this is borderline but the fix is cheap so I'll just push it" reasoning on a Medium that wouldn't bother you if you saw it in production
> - Cycling through 3 fix-commits chasing the bot's next style finding
> - Fixing a Medium "while I'm here" without a clear correctness reason
>
> **Lesson from practice:** a single review run once pushed three separate Medium-fix commits in one hour — each a type-polish nit, each triggering another bot round that surfaced the next cosmetic Medium. Only one arguably met the user-impact bar; the rest belonged in the backlog. The loop would have exhausted the daily review quota before the PR could proceed to Stage C. The criteria above are a floor (always fix when they hold) plus explicit judgment latitude beyond that — not a mandate to backlog real correctness fixes just because a test update is needed.
>
> **Reply template for backlog dismissals:** "Backlogged in docs/BACKLOG.md §\"<slug>\" — <brief rationale>. Scope appropriate for a follow-on PR; this PR already addresses the primary Critical/High concerns."

> [!IMPORTANT]
> **Pre-trigger subagent gate:** Before EACH bot-review trigger (auto or manual), confirm the most recent CLI-subagent run was on the current HEAD. If any commit was pushed after the subagent's last clean verdict, re-run the subagent before triggering the bot. The cost of one subagent run (~2 min, no quota) is dwarfed by a wasted bot round. This rule applies to EVERY fix cycle within Phase 14, not just the initial push.

**Update `releases-in-flight.md`:** Change status from `hardening` to `pr-opened` after the PR is created.

---

### Phase 15: SHA-Verified Exit Gate

Before advancing to Stage C, verify:
- The bot has reviewed the most recent **code-bearing SHA** (see doc-only escape below) — at least one bot review comment with `commit_id == LAST_CODE_SHA` and `created_at` after that push
- Zero open 🔴 critical findings on HEAD
- Every 🟡 non-critical has a reply (fix SHA or backlog-with-rationale)
- Every historical bot comment across all rounds has a reply
- No code-bearing pushes have occurred since the last bot auto-review

> [!IMPORTANT]
> **Doc-only escape:** The bot reviews code; doc-only pushes (BACKLOG.md, handoff.md, plan/spec drift fixes, known-patterns file, bot styleguide, cross-repo prompts, README updates, review-log entries) ride on the prior code-SHA's bot verdict. **Do NOT poll, wait for, or manually trigger the bot on doc-only pushes.** Each doc-only round-trip burns calendar time + review quota for zero risk-management signal.
>
> Strong signals a push is "doc-only":
> - Only files under `docs/`, `*.md`, your bot-config dir, or staging-input dirs
> - No source-code changes (`.ts` / `.js` / `.py` / native source / CI workflow `.yml`)
> - Pure comment changes inside source files (verify with `git diff --shortstat HEAD~1 -- '*.ts'` etc. → zero non-comment lines)
>
> Compute `LAST_CODE_SHA` as the most recent commit that touched any non-doc file. If `HEAD == LAST_CODE_SHA`, the strict check applies. Otherwise verify the bot has reviewed `LAST_CODE_SHA` (the code-bearing SHA) — doc commits after that pass through.

```bash
HEAD_SHA=$(git rev-parse HEAD)
PR_NUMBER=$(gh pr view --json number -q '.number')
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
BOT_LOGIN="<your-pr-review-bot-login>"   # e.g. the review bot's GitHub bot account

# Compute LAST_CODE_SHA — the most recent commit that touched non-doc files.
# A commit is "doc-only" iff every changed file is under docs/, ends in .md,
# or is under your bot-config / staging-input dirs. Adjust the regex to match
# your repo's documentation layout if needed.
LAST_CODE_SHA=$(git log --pretty=format:'%H' main..HEAD | while read sha; do
  if git show --name-only --format='' "$sha" 2>/dev/null \
     | grep -qvE '^(docs/|.*\.md$|inputs/|.*/CLAUDE\.md$)'; then
    echo "$sha"; break
  fi
done | head -1)
LAST_CODE_SHA="${LAST_CODE_SHA:-$HEAD_SHA}"

LATEST_BOT_SHA=$(gh api repos/$REPO/pulls/$PR_NUMBER/comments --paginate 2>/dev/null \
  | jq -r --arg bot "$BOT_LOGIN" '[.[] | select(.user.login==$bot)] | sort_by(.created_at) | last.commit_id // ""')

if [ "$LATEST_BOT_SHA" != "$LAST_CODE_SHA" ] && [ "$LATEST_BOT_SHA" != "$HEAD_SHA" ]; then
  echo "🔴 Bot has not reviewed the most recent code-bearing SHA ($LAST_CODE_SHA)."
  echo "   Latest bot-anchored SHA: $LATEST_BOT_SHA"
  echo "   Wait for review on $LAST_CODE_SHA, or push a bundled fix."
  exit 1
fi
echo "✅ Bot gate: HEAD=$HEAD_SHA last-code=$LAST_CODE_SHA last-bot=$LATEST_BOT_SHA"
```

**No duplicate re-audit.** The cross-model hardening skill's Push/PR/Bot phase already did the full SHA-pinned history audit. This is a single verification check, not a repeat audit.

---

## Stage C: Staging + Merge Readiness

The PR is bot-clean on HEAD. Stage C runs the post-review verification that requires infrastructure (staging) or human eyes.

### Phase 16: Rollback Plan (Documented)

23. Document in the PR description or your review log:
    - Exact rollback command(s) (e.g., `git revert HEAD~N`, specific service redeploy commands)
    - Database state recovery steps if applicable
    - Manual cleanup needed (e.g., clearing stale datastore fields)
    - For backend services: which functions/services are deployed and the exact deploy + rollback commands for each

---

### Phase 17: Auto-Doc Sync + Test Suite + Dep Scan

24. **Auto-Documentation Sync (MANDATORY before staging):**
    - Diff the branch and apply drift corrections to: `handoff.md`, `docs/BACKLOG.md`, `README.md`, the project constitution
    - Review `docs/QA_CASES.md` — add missing test cases for this changeset's edge cases

25. **Final test suite run** (Phase 4 ran once pre-push; re-run on the bundled HEAD):
    | Tech Stack | Command (example) |
    |-----------|---------|
    | Native mobile | framework test runner with coverage |
    | Native (compiled) | package test command |
    | Node.js | `npm test` |

26. **Dependency security scan:**
    | Tech Stack | Command (example) |
    |-----------|---------|
    | Node.js | `npm audit` |
    | Native mobile | dependency-outdated check |
    | Native (compiled) | review the resolved-dependencies lockfile |
    Critical/High vulns → must fix or document the exception. Moderate/Low → backlog.

---

### Phase 18: Staging Deploy / Build Verification

> **No production distribution happens in this phase.** Web/backend targets deploy to a true pre-prod **staging** environment for human QA. Native mobile does **not** distribute at all here — it only verifies the release build compiles; the merge → CI → release-channel distribution is the production-release workflow's job.

27. **Web / backend — deploy to staging (MANDATORY where a staging env exists):**
    - Consult the repo's project constitution for write-path classification.
    - All staging-capable targets MUST point to the **staging** environment for human QA before ANY production merge.
    - Platform-appropriate method:
      - **Web host:** push the feature branch → auto preview URL
      - **Static hosting:** `npm run deploy:staging` or equivalent
      - **Backend services:** deploy only the changed functions/services to the **staging** project
      - **Native mobile (where a separate staging distribution track exists):** build pointing to the **staging** backend (a separate staging distribution track, NOT production).
        - STOP and ask the human: "Please trigger the staging-distribution CI workflow pointing at the `feature/...` branch."
        - Then wait for the human to reply that staging QA is complete and approved. Do NOT merge to the trunk until then.
    - **Update `releases-in-flight.md`** status to `staging-deployed`.

27b. **Native mobile with no staging env — verify the release build, do NOT distribute:**
    - When a native mobile app has no separate staging environment and its release channel IS the real distribution, distribution is **deferred to the production-release workflow**.
    - In `/release-prep`, confirm the build compiles for release — the local build (Phase 4) + test run (Phase 17) already cover this. (Optional: if you've configured a **PR / build-only** CI workflow — one that builds + tests the PR branch WITHOUT a distribution action — let it run and check it green. The default merge-to-trunk CI workflow that distributes is NOT triggered here.)
    - **DO NOT** merge to the trunk. **DO NOT** trigger a CI distribution. **DO NOT** push to a release channel. Leave the bot-clean PR open and unmerged.
    - **Update `releases-in-flight.md`** status to `build-verified`.

---

### Phase 19: Automated AI Staging Regression

28. Always attempt to QA the staging environment before handing off to the human.
    - **Web / static hosting:** run a regression-test pass using a browser subagent against the generated staging URLs. Verify browser console logs and visual rendering.
    - **Mobile:** the AI cannot launch a staging distribution build. If the release touches the data layer or schema, write a temporary script using the backend admin SDK to ping the **staging** project. Verify data contracts and schema fallback resilience against live staging responses before human testing begins.
    - **Schema-change-triggered client regression:** if any datastore schema was touched (even if client code wasn't):
      1. Serve each client surface that consumes the schema (including older versions still in use) → verify rendering
      2. Load the primary detail/consumer view → verify data displays

29. **Database safety verification:**
    - If the write-path is greater than "no writes": verify the staging DB was used and no production writes occurred
    - For data-access-layer mutations: verify a dry-run was tested first

---

### Phase 20: Exit Gate — Human Approval

30. Present a summary to the human:
    - ✅ All Stage A hardening phases completed (with a one-line note per phase)
    - ✅ Bot-clean on HEAD SHA
    - PR # — **bot-clean and unmerged** (mobile-no-staging: build-verified locally, NOT distributed yet) / staging URL (web/backend)
    - Schema compatibility report (if applicable)
    - Compliance check results (if applicable)
    - Accessibility findings
    - Deferred items logged to the backlog

    > [!IMPORTANT]
    > **Low-blast-radius auto-chain (opt-in, per-repo):** for a repo where merging to the trunk carries **no** production blast radius — CI distribution is OFF, so merging does NOT build or distribute, and a separate **manual** archive/distribution step remains the real human gate — you may configure this workflow to skip the stop here. Once the exit gate is reached (bot-clean + build-verified), it sets `releases-in-flight.md` status to `approved` (with timestamp) and **immediately invokes the production-release workflow** — no waiting. Rationale: the merge is low-risk and the manual archive is the checkpoint. **All other projects STILL stop here** and wait for explicit go-ahead — the rule below applies to them.

    **🛑 STOP (default). Wait for the human's explicit go-ahead.** Their approval authorizes the production-release workflow to **merge the PR and distribute** (merge → CI → release channel). `/release-prep` itself leaves the PR unmerged and undistributed.

31. **Record approval in releases-in-flight.md (MANDATORY after the human says "go"):**
    Update the entry created in Phase 1:
    - Change `Status` from `staging-deployed` / `build-verified` to `approved`
    - Add an `**Approved:**` field with the current ISO8601 timestamp
    - This is the signal the production-release workflow's first phase checks for. Without it, the handoff gate aborts.
    ```markdown
    ### <repo> — <branch>
    - **PR:** #<number>
    - **Status:** `approved`
    - **Opened:** <date>
    - **Approved:** <timestamp>
    - **Workspace:** <pwd>
    ```

---

## Self-Improving QA Loop (MANDATORY — All Projects)

After every release-prep run, fold what you learned back into the system so the same class of finding never costs a full round again:

- A bot/reviewer finding that's likely to recur → add it to your known-patterns file (with the canonical fix) and your bot's styleguide.
- A missed edge case caught in staging → add a case to `docs/QA_CASES.md`.
- A gate that should have caught it but didn't → strengthen that gate's check.

This loop is shared with the production-release workflow. The discipline: every escaped defect becomes a permanent new check.

---

## Recovery / Re-Entry Points

If this workflow fails partway through, you don't need to re-run from Phase 0. Resume at:

| Failure point | Resume at |
|---------------|-----------|
| Build / lint / tests failed in Phase 4 | Fix, re-invoke the local pre-flight gate, continue at Phase 5 |
| Secrets / schema / compliance / a11y blocker | Fix, re-run the failed phase, continue |
| Cross-model convergence stuck after 4 rounds | Escalate to the human; do NOT push |
| Bot found Critical/High in Phase 14 | Return to Phase 12 on new HEAD, then Phase 14 again |
| Staging deploy failed | Fix the deploy config, resume at Phase 18 |
| Staging regression failed | Fix, re-push (triggers a bot round), resume at Phase 14 |
| Human requested changes at Phase 20 | Return to Phase 12; re-run through Phase 20 |

---

## Anti-Patterns

| Don't | Do Instead |
|-------|------------|
| Run compliance/schema/a11y AFTER the PR is open | Run pre-push (Phases 5–11) so fixes don't trigger bot rounds |
| Run the simplification pass after the first push | Run it in Phase 10, before cross-model convergence |
| Skip the local pre-flight gate because "I know it builds" | The gate catches docs drift and .env leaks too |
| Run the cross-model hardening pass separately after `/release-prep` | This workflow already runs those phases inline |
| Push incremental fix commits between bot rounds | Bundle the fixes — one push per bot round |
| Manually re-trigger the bot review | The bot auto-reviews every push. Manual triggers burn quota with no added signal |
| Treat bot silence as clean | Document as exceptional; proceed only with the human's explicit override |
| Declare converged without the SHA-pin check | The SHA-pin check is non-negotiable |
| Push a commit to "just fix this one Medium" after Criticals are cleared | Stop. Backlog it + reply with rationale. Each push burns a bot round, and the next round will find another Medium. See the Phase 14 STOP-AT-MEDIUMS block for the inline-fix criteria |
| Reason around the STOP-AT-MEDIUMS rule ("this fix is cheap, so one more round won't hurt") | The rule exists because EVERY push triggers a new review; quota is scarce. Trust the rule. Backlog. |
</content>
</invoke>
