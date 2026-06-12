---
name: peer-review
description: Autonomous code hardening pipeline. Completes unfinished work, runs adversarial cross-model review (a second model ↔ a CLI subagent), hardens QA cases, drives to convergence, then pushes to a feature branch, creates a PR, and runs the full PR-review-bot gate — all from a single invocation. Exits only with a bot-clean PR ready for staging release.
---

# /peer-review — Autonomous Release Hardening

> Install per [SETUP.md](../../SETUP.md). This skill is the operational, step-by-step home of Cross-Model Adversarial Review — it holds both the why and the how.

## Purpose

This workflow drives a codebase from "in progress" to "bot-clean PR" autonomously. It completes unfinished plan items, runs adversarial cross-model review loops, hardens the QA suite, then pushes to a feature branch, creates a PR, and runs the PR-review-bot gate until clean. Staging release (see [Staged Release](../../workflows/release-prep.md)) picks up from a verified-clean PR.

Throughout, **"the bot"** means your automated PR-review gate — a hosted AI code reviewer that auto-reviews PRs on push (e.g. Gemini Code Assist, GitHub Copilot review, or similar). The bot-gate logic below is the same regardless of which one you wire in.

> [!CAUTION]
> **SCOPE BOUNDARY:**
> - This workflow pushes to **feature branches only**. It NEVER pushes to `main` or `release`. It NEVER merges.
> - The PR is created for bot review, NOT for merging. Merging is the production-release step's job (after a human approves the bot-clean PR). The staging step itself stops at an approved, unmerged PR.
> - All internal hardening (cross-model convergence) MUST complete before pushing. The goal is a clean bot review on round 1.

**You are the Critic AND the Finisher.** Complete what's incomplete, then attack what's complete until you can't find anything wrong — and prove it with tools, not opinion.

> [!CAUTION]
> **NEVER declare convergence unless: (1) ZERO open Critical or Warning findings, (2) ALL safety properties verified with deterministic tools, (3) subagent loop has returned clean AND the bot has returned clean-or-Medium-only on the current HEAD.** Default to MORE testing, not less.

---

## Review Workflow Protocol (authoritative — supersedes any narrower phase-level rule)

**Stage 1 — Subagent loop (fast, unlimited):**
1. Launch a CLI subagent (fresh context) to adversarially review the current HEAD.
2. Fix every actionable Critical / High / blocking-Medium finding.
3. Re-push the updated HEAD.
4. Launch ANOTHER subagent on the delta.
5. Repeat until a subagent returns clean — zero Critical, zero High, no blocking Medium.

> [!CAUTION]
> **Subagent re-vet runs on EVERY commit pushed to the PR, no exceptions.** Even 1-line fix commits between bot rounds get a fresh subagent run on the new SHA. Skipping subagent re-vet on "small surgical fixes" once burned four bot rounds on a single line — each round caught something a fresh subagent run would have caught locally. SHA-pin invariant: a subagent verdict on SHA X is invalidated by ANY new commit; re-run on the new HEAD before triggering the bot.

**Required subagent prompt elements** (calibration learned the hard way):
- **"Default to flagging"** — not flagging is more costly than over-flagging once the bot is in the loop.
- **Calibration acknowledgment** — if a previous subagent on this PR called something "defensible" and the bot later escalated it, the next prompt must say: "Previous subagent runs have been wrong on this PR. Treat ANY ambiguity as a flag."
- **Naming-consistency check** against existing interfaces in the same file — a common escape is a serialized field that doesn't match its canonical interface name (e.g. `overall_score`/`tags_added`/`elapsed_ms` vs canonical `overall`/`tags`/`duration_ms`).
- **Field-by-field interface verification** when the diff includes a concrete artifact mirroring a typed interface (JSONL examples, JSON shapes, function signatures).
- **Cross-file consistency check** for any cited line ranges, SHAs, branch behind/ahead counts, or PR references that appear in multiple docs.

**Stage 2 — Bot gate:**
6. Trigger the bot's review on the subagent-clean SHA (or let it auto-review on push — see Phase 6).
7. Wait for the bot to post.

**Stage 3 — Severity-triaged bot disposition:**

- **Critical or High from the bot** → fix all, RETURN to Stage 1 (subagent must re-sign-off on new HEAD, then another bot round). Repeat until the bot returns clean-or-Medium.
- **Medium / Low / Info only** → fix Mediums per severity-triage below (inline if user-impact/blocker; backlog if optimization/defensive). Low/Info → backlog or accept. **DO NOT run another bot round.** Proceed directly to staging release.
- **Zero findings** → proceed directly to staging release.

**Severity-triage (for Medium+ findings during Stage 3):**
- Medium, fix inline if: user-facing failure, deploy-blocker risk, data corruption potential, styleguide violation with downstream impact.
- Medium, backlog if: optimization, defensive-but-not-required check, stylistic preference, minor observability.
- Low / Info: default backlog; fix inline only if trivial (<5 lines) AND in scope.

**A third reviewer is OPTIONAL, not in the standard loop.** A different frontier model (paste-and-wait, outside the CLI loop) has different blind spots from the subagents/bot and has caught things both missed. Invoke it manually as an escape hatch for high-risk PRs:
- Prod data mutation scripts
- Data-access-layer changes
- Auth / billing / compliance code paths
- Large-scope new features where cross-model coverage is worth the manual paste dance

For default PRs: subagent loop → bot → staging release → production release.

**Bot rate-limit discipline:** assume a finite budget of substantive bot reviews per PR per 24h (commonly ~3–5). Each Stage 2 trigger burns one. Running subagents to convergence FIRST means the bot's quota is spent on genuine catches, not lint-level churn.

**SHA-pin discipline:** the subagent that signs off in Stage 1 and the bot review in Stage 2 MUST be on the same SHA. If a fix is pushed after subagent sign-off, re-run a subagent delta-check before triggering the bot. A subagent's clean verdict on a stale SHA does NOT count.

Protocol evolution (the lesson, not the dates):
- Original: exhaust the second model + a third reviewer before the bot (worked but friction-heavy).
- Refined: all reviewers clean on the exact SHA before the bot (tighter SHA discipline).
- Current: the third reviewer moved out of the standard loop; subagents + bot are the baseline; the third reviewer is on-demand for high-risk PRs.

---

## Critical Principles

1. **Context Isolation:** Bootstrap ONLY from disk artifacts: the state/handoff doc, specs, git diffs, `QA_CASES.md`, your bot's sacred-rules file. Never read conversation logs — they inherit the builder's reasoning biases.
2. **Deterministic Over Opinion:** Use test runners, builds, linters, and curl. Your judgment is the LAST resort.
3. **Evidence-Based Findings:** Every finding MUST cite file, line, and a concrete failure scenario. No "this could be a problem."
4. **Bounded Remediation:** Stay within the blast radius. No style refactors. No feature additions. Fix what's broken — nothing more.
5. **Thoroughness Over Speed:** Read every changed line. Trace every code path. For diffs >500 lines, chunk and review independently.
6. **Structural Changes Invalidate Everything:** If you change a guard, safety boundary, or core mechanism, ALL prior verifications are void. Re-verify from scratch.
7. **No Premature Convergence:** Two consecutive clean passes from different models. If you fixed anything, you MUST recommend another round. Non-negotiable.

---

## Phase -2: BOT-DEBT RESOLUTION

**Goal:** Before starting any new review work, check the current repo's open PRs for unresolved bot findings. If any exist, fix them via a CLI subagent, then **restart the entire `/peer-review` workflow from scratch.** This ensures every fix goes through the full hardening pipeline.

> [!IMPORTANT]
> This phase operates on the CURRENT REPO ONLY. It checks open PRs for the branches being worked on in this workspace.

> [!CAUTION]
> **RESTART RULE:** If this phase applies ANY fixes, the entire `/peer-review` workflow restarts from Phase -2. No skipping ahead. The fixes themselves are code changes that must go through the full pipeline (Phases -1 through 7). This is non-negotiable.

### Step -2.1 — Discover open PRs with bot comments

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')

# List all open PRs in this repo
gh pr list --repo $REPO --state open --json number,headRefName,title \
  --jq '.[] | "\(.number) \(.headRefName) \(.title)"'
```

If no open PRs exist → skip to Phase -1.

### Step -2.2 — Scan each PR for unresolved bot comments

For each open PR, fetch the review-bot's comments. Substitute your bot's GitHub login for `<bot-login>` (e.g. `gemini-code-assist[bot]`, `copilot-pull-request-reviewer[bot]`):

```bash
BOT_LOGIN='<bot-login>'
for PR_NUM in $(gh pr list --repo $REPO --state open --json number -q '.[].number'); do
  echo "=== PR #$PR_NUM ==="

  # Issue comments (summary reviews)
  gh api repos/$REPO/issues/$PR_NUM/comments --paginate \
    | jq --arg b "$BOT_LOGIN" '[.[] | select(.user.login==$b)]'

  # Inline review comments (code-level findings)
  gh api repos/$REPO/pulls/$PR_NUM/comments --paginate \
    | jq --arg b "$BOT_LOGIN" '[.[] | select(.user.login==$b)]'
done
```

For each PR with bot comments:
1. Fetch the full comment bodies (file, line, description, suggested fix)
2. Identify findings that are **unresolved** — no reply with a fix commit SHA or dismissal rationale
3. Cross-reference against the current HEAD of that PR's branch to determine if the finding still applies

**If zero unresolved bot findings across all PRs → skip to Phase -1.**

### Step -2.3 — Fix unresolved findings via a CLI subagent

For each PR with unresolved findings, checkout the branch and delegate fixes. The dispatch shape below is generic — a fresh CLI subagent, given a model + effort flag, writing its report to an output file you read back. Adapt the command to your harness:

```bash
# Save current branch for later
ORIGINAL_BRANCH=$(git branch --show-current)

# Stash any uncommitted work
git stash --include-untracked

# Checkout the PR's branch
git checkout <branch-name>
git pull origin <branch-name>

# Delegate fixes to a fresh CLI subagent (generic dispatch — adapt to your harness).
# Pattern: fresh subagent, model + effort flags, read the report from an output file.
AGENT_OUTPUT_FILE=/tmp/bot-debt-fix-PR<number>.md \
run-cli-subagent \
  -p "You are resolving unresolved PR-review-bot findings on PR #<number> in $(pwd).
Branch: <branch-name>. Read the bot's sacred-rules file for repo rules.

UNRESOLVED BOT FINDINGS:
<paste each finding with file, line, and description>

For each finding:
1. Read the cited file and surrounding context
2. Apply the fix if legitimate (cite the finding in the commit message)
3. If the finding is invalid, document WHY with a concrete rationale
4. Run tests/build/lint after all fixes
5. Output a structured report: finding → resolution (fixed or dismissed with rationale)

Do NOT refactor for style. Do NOT touch code outside the cited findings." \
  --model <capable-tier> --effort high --output-format text
```

### Step -2.4 — Review the subagent's fixes and push

Read `/tmp/bot-debt-fix-PR<number>.md`. For each fix:
1. **Verify independently** — check the file and confirm the fix is correct
2. Classify: AGREE / PARTIAL (apply better fix) / DISAGREE (revert)
3. Run tests/build/lint to confirm nothing broke
4. Reply to each resolved bot comment on the PR with the fix commit SHA or dismissal rationale

Push the fixes:
```bash
git push origin <branch-name>
```

### Step -2.5 — RESTART THE ENTIRE WORKFLOW

**After applying fixes to ANY PR, return to the original branch and restart:**

```bash
git checkout $ORIGINAL_BRANCH
git stash pop  # Restore any stashed work
```

**→ Go back to Phase -2, Step -2.1.** The full workflow runs again from scratch. Phase -2 will re-scan and confirm the debt is cleared. The rest of the pipeline (Phases -1 through 7) will harden the fixes through cross-model review and the bot gate.

> [!WARNING]
> **Do NOT skip the restart.** Bot-debt fixes are code changes. Code changes require full pipeline hardening. Proceeding directly to Phase -1 without restarting would bypass the verification that the fixes themselves are correct and don't introduce new issues.

---

## Phase -1: COMPLETE UNFINISHED WORK

**Goal:** Check if the plan/spec has unimplemented items. If so, execute them before reviewing.

> [!IMPORTANT]
> This phase only executes ALREADY PLANNED AND APPROVED work. You are not designing new features — you are finishing what the previous session started.

### Step -1.1 — Identify incomplete items

Read the state/handoff doc and the implementation plan (spec or design artifact). List items marked as incomplete, TODO, or "next steps."

### Step -1.2 — Validate the plan

For each incomplete item, verify it still makes sense given the current code state. If an item is outdated or conflicts with completed work, note it and skip. If the plan has gaps or errors, improve it and document what you changed and why.

### Step -1.3 — Execute

Implement each remaining item following existing code patterns. Commit atomically per concern.

> [!NOTE]
> **Execution method is mandated by Step -1.3a below.** Do NOT implement items inline in your own coordinating context. Each enumerated item is dispatched to a fresh subagent per the subagent-driven execution protocol. (This is the EXECUTION-side analog of the review-side subagent loop in the top-of-file "Review Workflow Protocol" — they are separate loops and do not replace each other.)

#### Step -1.3a — Subagent-driven execution (fresh subagent per task)

*(adapted from [obra/superpowers](https://github.com/obra/superpowers) `subagent-driven-development`, MIT; mirrors the [AI dev lifecycle](../../concepts/ai-dev-lifecycle.md) execute phase)*

> [!IMPORTANT]
> **This is the EXECUTION loop. It does NOT touch the review-side subagent machinery.** The top-of-file "Review Workflow Protocol" (Stage-1 subagent re-vet on every SHA), Phase 5 (cross-model convergence), and Phase 6 (bot gate) all run UNCHANGED, AFTER Phase -1 finishes. A subagent here BUILDS a task; a subagent there REVIEWS a pushed SHA. Two different loops. Never collapse one into the other.

**Dispatch a fresh subagent per task.** For each incomplete item enumerated in Step -1.2, spawn a NEW CLI subagent with isolated context. Hand it the task's full text — do NOT make it read the whole plan. Construct exactly the context it needs (the spec clause, the target file paths, the bot's sacred-rules file, the relevant existing pattern). Keeps the coordinating context clean and the subagent focused.

**Per-task two-stage review — spec-compliance THEN code-quality, in that order:**
1. **Implement** — search existing code first, match patterns, write the failing test → minimal code → run it. (Builder subagent.)
2. **Stage A — Spec-compliance review** (a SEPARATE reviewer subagent): does the implementation match the spec clause from Step -1.2 — nothing missing, nothing extra, no scope creep? Fix → re-review until clean. Runs FIRST and is gating.
3. **Stage B — Code-quality review** (a second, distinct reviewer subagent): **only after Stage A passes.** Correctness, edge cases, conventions, defensive parsing (mirror Phase 3's pre-push sweeps). Fix → re-review until clean.
4. **Lint / typecheck, mark done, commit atomically** (one commit per concern, per Step -1.3).

Running quality review before spec review wastes capable-model budget polishing code that may be solving the wrong thing — spec-compliance is the cheaper gate and runs first.

**Status protocol — every subagent returns exactly one terminal status. Handle each honestly; NEVER silently retry.**

| Status | Meaning | Coordinator action |
|---|---|---|
| `DONE` | Task complete, both review stages clean, committed. | Advance to next task. Do NOT pause for "should I continue?" — execute the plan. |
| `DONE_WITH_CONCERNS` | Committed, but flagged a non-blocking risk. | Accept the commit. Log the concern to `docs/BACKLOG.md` with task ref. Advance. |
| `NEEDS_CONTEXT` | Subagent lacked info to finish correctly. | **Something must change.** Supply the missing context and re-dispatch a FRESH subagent — never re-prompt depleted context. If it's a plan defect, fix the plan per Step -1.2. |
| `BLOCKED` | Cannot proceed (failing dependency, contradiction, decision above pay grade). | **Something must change** — stronger model, smaller task, or human escalation. Do NOT retry blindly. Surface to the human if it needs a decision. |

`NEEDS_CONTEXT`/`BLOCKED` mean an input was wrong, not that the subagent failed. A silent retry on the same context is the failure mode this protocol prevents.

**Model-tiering — match model cost to task character:**

| Task character | Model | Examples |
|---|---|---|
| **Mechanical** | cheap tier | rename, move, format, mechanical refactor, boilerplate scaffolding, known-transform fan-out |
| **Design / review** | capable tier (high effort) | new logic, interface design, edge-case reasoning, BOTH review stages |

Reviewer subagents are ALWAYS capable-tier regardless of how mechanical the build was. Dispatch shape mirrors the Phase 5 Step 5.2 / Phase -2 Step -2.3 calls (a fresh CLI subagent with `--model <tier> --effort <level> --output-format text`, reading the report back from `AGENT_OUTPUT_FILE`).

**Checkpoint discipline:** a pushed commit is the only proof a task is done. Commit per task; if the session dies mid-plan, the next `/peer-review` re-enters Phase -1 and committed tasks are already off the list.

> [!CAUTION]
> When ALL items return `DONE`/`DONE_WITH_CONCERNS`, proceed to **Step -1.4 — Verify implementation** (full suite) UNCHANGED. Per-task Stage-A/B reviews do NOT replace Step -1.4's whole-suite verification, and Phase -1 completion does NOT exempt the resulting commits from the review pipeline — they still flow through Phase 0 → Phase 6 like any other diff.

### Step -1.4 — Verify implementation

Run the full verification suite (tests, build, lint) after completing all items. Fix any failures before proceeding.

**If no unfinished items exist, skip to Phase 0.**

### Step -1.5 — Bootstrap pattern-learning infrastructure (one-time per repo)

Ensure these two files exist. They're append-only learning surfaces that reduce bot churn over time:

- **Your known-patterns file** (e.g. `docs/.known-patterns.md`) — known recurring bot findings with canonical fix / dismissal rationale. Phase 3 step 5 greps the diff against this. Phase 6 Step 6.4 appends to it. If missing, create with:
  ```markdown
  # Known Code-Review Patterns

  Append-only list. Each entry: CATEGORY · pattern · canonical fix OR canonical dismissal rationale. Used by `/peer-review` Phase 3 (pre-push grep) and Phase 6 (dismissal paste-lookup).
  ```
- **Your bot's styleguide** (the file your PR-review bot honors on review — e.g. a `styleguide.md` in the bot's config directory) — rules the bot honors on review. Teaches the bot the repo's conventions upfront so findings don't need post-review dismissal. If missing, create with a minimal header and add rules as you find ones the bot should honor:
  ```markdown
  # Code Review Style Guide

  Rules for the PR-review bot when reviewing PRs in this repo. Conventions the bot should honor without re-flagging.
  ```

Both files are repo-local — no platform assumptions. Create both if missing; don't delete content if they exist.

---

## Phase 0: SAFETY PROPERTY ENUMERATION

**Goal:** Before reading code, enumerate what MUST hold true. This prevents tunnel vision.

### Step 0.1 — List safety-critical properties

Based on the business goal, spec, and file types in the diff:

```
SAFETY PROPERTIES:
1. [Property]: [What must be true] — [How to verify deterministically]
```

Examples: No infinite loops, no data loss, no cost explosion, no schema drift, no auth bypass.

### Step 0.2 — Verify each independently

Trace each property end-to-end. Use `grep`, schema file checks, and code path tracing — not "it looks right."

---

## Phase 1: ORIENT (Read-Only)

**Goal:** Bootstrap context from disk artifacts. Understand what was built, why, and what "done" looks like.

### Step 1.1 — Read repo instruction files

```
1. The bot's sacred-rules file — sacred rules, deploy constraints, gotchas
2. The state/handoff doc — previous session state
3. docs/specs/*.md — spec documents for current work
4. docs/BACKLOG.md — current backlog
5. docs/QA_CASES.md — existing QA test cases
6. docs/peer-review-log.md — previous review rounds
```

### Step 1.2 — Identify business goal (anti-paraphrase)

The single most common source of drift is paraphrasing the user's stated goal across sessions until the implementation addresses a paraphrased version rather than the original. This step prevents that.

Emit ALL of the following — no shortcuts:

```
GOAL (VERBATIM): "<direct quote from the user's originating ask — PR body,
  issue, SPEC file, or linked conversation. NO paraphrase. If you have to
  summarize, state that explicitly and explain why the verbatim isn't available.>"

MY INTERPRETATION: "<1-2 sentence restatement of what the goal actually
  commits to, in concrete testable terms. If this diverges from the verbatim,
  flag the divergence inline.>"

WHAT THIS PR DELIVERS: "<1 sentence describing the diff's observable effect>"

GOAL ↔ DELIVERY MAPPING: "<1 sentence proving the delivery matches the goal,
  OR flagging a gap.>"

RED TEAM: "<What could this PR silently violate about the goal that a
  line-by-line code review wouldn't catch? Cite specific file:line concerns
  OR state explicitly: 'No plausible path to goal violation identified.'>"
```

**If you cannot emit all 5 paragraphs with specifics, HALT the review and ask the user.**

The review is not complete until Goal Alignment (Phase 2 dimension #2) is formally checked against this statement. A real drift incident shows why: a "thin wrapper = parity" paraphrase rode through seven PRs before a human caught it. This 5-paragraph discipline catches that class of drift at review time.

### Step 1.3 — Map the change set

```bash
git branch --show-current
git diff main --stat && git diff main --name-only
```

Review the actual diffs — this is your primary review surface.

### Step 1.4 — Identify QA surface

Cross-reference `docs/QA_CASES.md` against changed functionality. Note gaps where no QA case covers a changed code path.

### Step 1.5 — Audit write paths

For every datastore write in the diff:
1. List every field being written
2. Verify field exists in schema validators
3. Verify field exists in write-guard / known-fields
4. Verify write includes loop-prevention flags where applicable
5. Document gaps as 🔴 findings

---

## Phase 2: ANALYZE (Read-Only — Produce Findings)

**Goal:** Review the change set against six dimensions. Produce severity-classified findings.

### Review Dimensions

| # | Dimension | What to check |
|---|-----------|---------------|
| 1 | **Correctness** | Logic bugs, null paths, off-by-one, race conditions, unhandled rejections, wrong return types |
| 2 | **Goal Alignment** ⭐ | Compare implementation to the Step 1.2 verbatim goal. See ⭐ deep-dive below — REQUIRED before any finding can be classified Critical/Warning. |
| 3 | **Security** | Input validation, auth bypasses, secrets exposure, injection vectors |
| 4 | **Edge Cases** | Empty arrays, null inputs, concurrency, timezones, network failures, large data |
| 5 | **Convention Compliance** | The bot's sacred rules, established patterns (data-access layer, etc.), naming, consistency |
| 6 | **Data Safety** | Data corruption, infinite loops, unbounded costs, stale-over-fresh writes |

#### ⭐ Goal Alignment deep-dive (MANDATORY)

Using the Step 1.2 verbatim goal statement as the ONLY input, answer:

1. **Paraphrase check.** Does any PR description, commit message, or spec claim use words the verbatim goal does NOT contain (e.g., "thin wrapper", "parity", "same as X") to describe the change? If yes: verify the paraphrase holds byte-for-byte or flag as 🔴 (the canonical lesson: "thin wrapper over buildUpdate" was TRUE but was used to imply "parity with the other path" which was FALSE — dozens of invariant drifts followed).

2. **Stated-vs-implemented contract check.** List every concrete promise in the verbatim goal (schema field, invariant, numeric threshold, behavioral guarantee). For each: cite file:line in this PR's diff that delivers it, OR flag as 🔴 "goal clause not delivered."

3. **Negative-space check.** What does the verbatim goal EXCLUDE or forbid ("no emails", "no cloud APIs", "no new writes")? For each exclusion: verify the PR diff doesn't introduce it, OR flag as 🔴 "goal constraint violated."

4. **Cross-path consistency check.** If the goal says "X and Y should do the same thing" (common pattern: live path vs batch path, mobile vs web): grep both paths in this PR + dependent code. Cite the shared primitive or co-validating test that enforces consistency, OR flag as 🔴 "cross-path claim unverified."

Classify any finding here as **🔴 Critical** by default — goal drift is not a Suggestion, it's a broken spec.

### Severity Classification

| Severity | Criteria | Action |
|----------|----------|--------|
| 🔴 **Critical** | Correctness bug, security vuln, data loss, goal not achieved, sacred rule violation | Fix immediately in Phase 3 |
| 🟡 **Warning** | Missing edge case, incomplete error handling, test gap | Fix with justification in Phase 3 |
| 💭 **Suggestion** | Style preference, optimization, nice-to-have | Log to `docs/BACKLOG.md` only |

### Anti-Hallucination Gate

Before classifying any finding as 🔴 or 🟡, ALL must be true:
- [ ] Exact file and line number cited
- [ ] Concrete failure scenario described
- [ ] Verified it's not handled elsewhere (upstream validation, middleware, try/catch)
- [ ] Not pattern-matching against a generic mistake that doesn't apply here

If any is "no" → downgrade to 💭 or discard.

### Judge Pass (Self-Filtering)

1. Remove duplicates describing the same underlying issue
2. Remove contradictory findings (uncertain analysis — discard both)
3. Remove findings about unchanged code (outside blast radius)
4. Verify 🔴 findings have concrete failure scenarios
5. If >10 findings remain, you're likely too noisy — re-evaluate and tighten

---

## Phase 3: REMEDIATE & VERIFY

**Goal:** Fix 🔴 and 🟡 findings. Prove fixes correct. Max 3 iterations.

### Rules of Engagement

- **🔴 Critical:** Fix immediately. No permission needed.
- **🟡 Warning:** Fix with inline comment. If >20 lines or architectural, ASK user first.
- **💭 Suggestion:** Log to `docs/BACKLOG.md`. Do NOT modify code.
- **Max scope:** 5 files per pass. Split if more need changes.
- **Every fix must be independently correct.** No fix should depend on another uncommitted fix.

### Verification Hierarchy

| Priority | Method |
|----------|--------|
| 1 | Run existing tests (`npm test`, `flutter test`, `swift test`, etc.) |
| 2 | Run build (`npm run build`, `tsc --noEmit`, `flutter build ios --no-codesign`, etc.) |
| 3 | Run linter at **MAXIMUM STRICTNESS** (see table below) |
| 4 | HTTP/curl check (if API endpoints affected) |
| 5 | Manual code path trace (last resort) |

**Strict linter commands by platform** (use these, not the defaults):

| Platform | Strict Command | Why |
|----------|---------------|-----|
| Flutter/Dart | `dart analyze` + `flutter analyze --fatal-infos` | Treats info-level findings as failures — catches what the bot catches |
| Swift | `swiftlint lint --strict` (if configured) + `swift build -Xswiftc -warnings-as-errors` | Warnings become errors — no "it's just a warning" escapes |
| Node/TypeScript | `eslint . --max-warnings=0` + `tsc --noEmit --strict` | Zero warnings policy — the bot flags every one of these |
| Python | `ruff check . --select ALL` or `flake8 --max-line-length=120` | Broadest rule set |

### Iteration Protocol

```
FOR iteration IN 1..3:
    1. Run all verification methods
    2. ALL pass → proceed to Phase 4
    3. Failures?
       a. Same errors as last iteration? → STOP. Escalate to human.
       b. New errors → analyze, apply targeted fix, increment counter
END FOR
After 3 failed iterations → STOP. Report stuck state. Human decides.
```

### Pre-push hardening rules (prevent bot rounds)

The bot is pattern-driven. Every finding caught locally is one bot round you don't need. Before advancing to Phase 4 (or at latest, before pushing in Phase 6), run these five sweeps on the working tree:

**1. Fan-out the fix.** After applying any code fix, grep the SAME pattern across the repo:
```bash
# Extract the pattern from the fix, then find unfixed siblings:
grep -rn "<pattern>" lib/ packages/ src/    # adjust paths per repo
```
If the fix is correct for the cited line, it's correct everywhere the pattern appears. Apply the same transform to all sites.

**2. Defensive-parsing sweep** (anywhere the diff consumes external/backend data). Grep for unchecked casts:
| Platform | Risky | Safer |
|---|---|---|
| Dart | `as List`, `as Map`, `as String` without `?` | `is List<dynamic>` type-guard then cast |
| Swift | force-unwrap `!` or `as!` | `guard let … = … as? Type` / `if let` |
| TypeScript | `as X`, `x!.y`, `any`, untyped `JSON.parse` | Zod validators, type predicates, discriminated unions |
| Python | `dict["key"]` without `.get(...)` on possibly-missing | `.get("key", default)` or pydantic |

Any external data access that THROWS on schema drift becomes a 🔴 critical under the new convention.

**3. Wrapper-property audit.** If the diff adds code that reads `x.foo` on a wrapper type (a type that `implements` / conforms to / extends another), open the wrapper class and verify `foo` delegates correctly. Wrappers that silently return defaults (`[]`, `null`, `""`) instead of delegating are a common subtle-bug source exposed by new consumers.

**4. Strict analyzer pre-push.** Even if CI is relaxed, the pre-push local check MUST be strict (table above). The 40-info lint diff is a pattern mine for bot findings.

**5. Pattern bank check.** Load your known-patterns file and the bot's styleguide (if present). For each pattern listed, grep the diff. Every match is a finding the bot will post — fix it before pushing.

These sweeps take minutes and cut bot rounds materially. Do them.

---

## Phase 4: QA HARDENING

**Goal:** Grow the regression suite. Prioritize autonomous (machine-runnable) test cases.

### Step 4.1 — Review `docs/QA_CASES.md`

For every finding (fixed or not) in this review, determine if a QA case exists that would catch it. If not, add one.

### Step 4.2 — Add autonomous test cases

Prioritize test cases that can be verified WITHOUT human interaction:

| Type | Examples | Automation |
|------|----------|------------|
| **Unit tests** | Function correctness, edge cases, error handling | Test runner (`npm test`, `flutter test`, `swift test`) |
| **Build verification** | Compilation, type checking, import resolution | Build command |
| **Lint/static analysis** | Convention compliance, dead code, type safety | Linter |
| **API contract tests** | Endpoint response codes, schema validation | `curl` or HTTP client |
| **Browser smoke tests** | Page loads, critical element presence, no JS errors | Browser subagent |
| **Schema validation** | Field existence in validators, write-guard coverage | `grep` / file parsing |

### Step 4.3 — Tag automation level

Every QA case in `docs/QA_CASES.md` should be tagged:
- `[AUTO]` — Can be run by AI without human (build, tests, curl, browser agent)
- `[MANUAL]` — Requires human action (physical device, visual sign-off, gesture testing)
- `[SEMI-AUTO]` — AI can run but needs human to verify result (screenshot comparison)

### Step 4.4 — Run all `[AUTO]` cases

Execute every `[AUTO]`-tagged QA case. Report results. Fix any failures (returning to Phase 3 rules).

> [!IMPORTANT]
> The QA suite must be STRONGER after every `/peer-review` invocation. If you can't add at least 2 new test cases, you haven't looked hard enough.

---

## Phase 5: CONVERGENCE LOOP (CLI Cross-Model Review)

**Goal:** Achieve two consecutive clean passes from different AI models. Automate the second-model ↔ CLI-subagent review cycle.

### Step 5.1 — Self-review checkpoint

After your own Phases 0–4 are complete, assess:
- Did you fix any 🔴 or 🟡 findings? → Cross-model review is MANDATORY.
- Zero findings found? → Still delegate to a CLI subagent for one cross-model pass (different blind spots).

### Step 5.1b — Prefer a third reviewer before the bot when it pays

When the bot's quota is a limiting factor OR when the diff contains a new user-facing feature, request a third-reviewer peer-review BEFORE opening the PR (Phase 6). Reasons:
- A third frontier model has caught structural issues the bot missed (operation-discriminator ambiguity; a video-scheme-bypass XSS) — unknown blind spots.
- Fixes the third reviewer surfaces land as one bundled commit in the PR-opening push, which means the bot's auto-review fires on fully-hardened code — cleanest single-round.
- Preserves bot quota for the final gate rather than burning it on code you know has issues.

Pattern:
1. Prepare the diff locally (Phases 0-4).
2. Write the third-reviewer prompt (spec context + SHA + what you want verified + anti-hallucination gate instruction).
3. Hand the prompt to the human; wait for them to paste the resolved review.
4. Adjudicate findings (Phase 5.3 rules below).
5. Bundle fixes + re-run the CLI cross-model pass (Phase 5.2).
6. If any third-reviewer round surfaces NEW findings, run another round on the fixed SHA — continue until it comes back clean.
7. Only THEN push + open PR (Phase 6), which triggers the bot.

Manual bot-trigger comments remain an anti-pattern — let the bot auto-review on push.

### Step 5.2 — Delegate to a CLI subagent

```bash
AGENT_OUTPUT_FILE=/tmp/peer-review-round-N.md \
run-cli-subagent \
  -p "Adversarial code review of $(git branch --show-current) in $(pwd). Business goal: {goal}. Base: main. Read the bot's sacred-rules file for repo rules. Run git diff main, review all changes against: Correctness, Goal Alignment, Security, Edge Cases, Conventions, Data Safety. Cite file+line+failure scenario for each finding. Apply anti-hallucination gate. Run tests/build/lint. Fix 🔴/🟡 findings. Output structured report. Do NOT refactor for style." \
  --model <capable-tier> --effort high --output-format text
```

### Step 5.3 — Parse and adjudicate the subagent's output

Read `/tmp/peer-review-round-N.md`. For each finding:
1. **Verify independently** — check the cited file and line yourself.
2. **Classify:** AGREE (fix) / PARTIAL (valid finding, better fix) / DISAGREE (dismiss with rationale)
3. Fix agreed findings per Phase 3 rules.

### Step 5.4 — Re-verify after fixes

Run full verification suite. If fixes introduced new failures, iterate (Phase 3 protocol).

### Step 5.5 — Convergence check

```
IF (the subagent found 0 Critical/Warning findings AND this model found 0 in latest pass):
    → TWO CONSECUTIVE CLEAN PASSES. Proceed to Step 5.6 (Bot Simulation).

ELIF (the subagent found issues that were fixed):
    → Delegate ANOTHER round to a fresh subagent (increment N).
    → Max 4 total cross-model rounds. After 4, escalate to human.

ELIF (the subagent and this model disagree on findings):
    → Use deterministic verification to break the tie.
    → If no deterministic answer, escalate to human.
```

> [!IMPORTANT]
> Every round that introduces fixes invalidates the previous "clean" status. The counter resets. Two CONSECUTIVE cleans from DIFFERENT models is the only exit criterion.

> [!CAUTION]
> **ANTI-SKIP GUARD — READ THIS BEFORE PROCEEDING**
> At this point, cross-model convergence is complete but **THE BOT HAS NOT RUN YET.**
> You are NOT done. You CANNOT claim "the bot reports zero findings" or "bot clean" or any equivalent.
> The bot is a SEPARATE, EXTERNAL service that reviews the PR on GitHub. It runs ONLY in Phase 6.
> Cross-model convergence (second model ↔ CLI subagent) is NOT the same as the bot gate.
> **If you skip Phase 6 and report the bot clean, you are hallucinating process — the most dangerous failure mode.**

### Step 5.6 — Bot Simulation Pass (MANDATORY before pushing)

> [!IMPORTANT]
> **This step exists because cross-model review and the bot have different blind spots.** Cross-model review checks correctness and architecture. The bot checks pedantic code hygiene. This step bridges the gap.

After convergence (two clean cross-model passes), run a targeted bot simulation:

1. **Load your known-patterns file AND the bot's styleguide** — read every pattern / rule. The known-patterns file records past bot findings (what to grep for). The styleguide holds conventions the bot already honors (violations there are definitely going to be flagged).

2. **Mechanically check each pattern against the diff:**
   ```bash
   git diff main --unified=0 | head -2000   # Review all changed lines
   ```
   For EVERY pattern in the known-patterns file, grep or visually confirm it does not appear in the diff.

3. **Run a bot-style pedantic review via a CLI subagent:**
   ```bash
   AGENT_OUTPUT_FILE=/tmp/bot-simulation.md \
   run-cli-subagent \
     -p "You are a pedantic static code analyzer reviewing $(git diff main) in $(pwd).
   Read the known-patterns file for known bot patterns.
   For EVERY changed line, check these categories:
   1. Assertions/expects without descriptive failure messages
   2. Shared state mutations outside lock/sync scope
   3. Types less precise than provable (unnecessary Optionals, Any, dynamic)
   4. Duplicated logic across methods (>5 LOC overlap)
   5. Unused imports, dead code, console.log/print in production paths
   6. Generic error messages instead of contextual ones
   7. Inefficient encoding/serialization in constrained contexts
   8. Missing input validation on public APIs
   9. Any pattern listed in the known-patterns file
   Output ONLY findings in format: [FILE:LINE] [CATEGORY] [DESCRIPTION]. No praise, no summary." \
     --model <capable-tier> --effort high
   ```

4. **Review simulation output:** Read `/tmp/bot-simulation.md`. For each finding:
   - Verify against the actual code
   - Fix legitimate findings (Phase 3 rules)
   - If ANY fixes made → convergence is void, re-run Step 5.5 (cross-model) first

5. **Only proceed to Phase 6 (Push/PR/Bot) when the bot simulation returns zero findings.**

> [!CAUTION]
> **PHASE 5 IS COMPLETE. THE BOT HAS STILL NOT RUN.**
> Everything above was LOCAL hardening (cross-model review + bot simulation).
> The REAL bot has not seen this code yet. It runs in Phase 6 when you push and create a PR.
> Do NOT write "the bot reports zero findings" or "bot clean" in any output until Phase 6.5 SHA-VERIFIED CLEAN GATE passes.
> Violating this is a sacred-rule-tier failure — it deceives the human about the actual state of the pipeline.

---

## Phase 6: PUSH, PR & BOT GATE

**Goal:** Push the feature branch, create a PR, and run the PR-review-bot gate until clean. This is the ONLY place the bot runs — no ad-hoc loops anywhere else.

> [!CAUTION]
> **This phase runs ONLY after Phase 5 convergence (two consecutive clean cross-model passes).**
> All internal hardening must be done BEFORE pushing. The goal is a clean bot review on the first try.
> If the bot finds issues that require fixes, you MUST re-run Phase 5 convergence after fixing (fixes invalidate prior clean passes).

### Step 6.1 — Pre-push verification

```bash
# Verify identity and branch
git config user.email          # Must match the repo's identity requirements
git branch --show-current      # Must be a feature branch, NOT main or release
git remote -v                  # Verify correct remote
```

If on `main` or `release` → 🔴 STOP. You cannot push from here.

### Step 6.2 — Final pre-push pattern scan

Load your known-patterns file AND the bot's styleguide. For each pattern:
- Grep the diff
- If match, apply the canonical fix from the pattern entry

This is the last check before the bot auto-reviews your push. Every pattern you catch here saves a full bot round.

If you added new patterns in Phase 3 (fan-out, wrapper audit, defensive parsing), append them to your known-patterns file before pushing so the next run inherits the learning.

### Step 6.3 — Push & create PR

```bash
BRANCH=$(git branch --show-current)
git push origin $BRANCH
```

**Wait 5 seconds**, then check if a PR already exists:
```bash
EXISTING_PR=$(gh pr view --json number -q '.number' 2>/dev/null)
```

- If PR exists → use it. Do NOT create a duplicate. **If it's a draft, mark it ready for review** (`gh pr ready <num>`) — many PR-review bots skip draft PRs by default, so drafts block the whole bot gate.
- If no PR → create one:
  ```bash
  gh pr create --fill    # Do NOT pass --draft; see note below.
  ```

> [!IMPORTANT]
> **Do NOT create the PR as a draft.** Many PR-review bots skip draft PRs by default, which means none of the auto-reviews this skill depends on will fire. A real incident: a PR sat in draft after creation, no bot review ever posted, and minutes of silent polling burned before the cause was identified.
>
> The PR is created as ready-for-review so the bot auto-reviews every push. Downstream staging runs its own compliance checks on the already-reviewed PR; `ready` state is the correct resting point during peer-review.

### Step 6.4 — Bot Gate (auto-review driven, no manual trigger)

> [!IMPORTANT]
> **The bot auto-reviews every PR and every push.** Most hosted PR-review bots fire automatically on PR open and on every subsequent push. You do NOT manually post a trigger comment — that only burns bot quota without adding signal.
>
> The levers for controlling review volume are:
> 1. **Make the first push good.** Finish all internal hardening (Phase 0–5) before pushing the PR-opening commit.
> 2. **Bundle fix commits.** Between bot rounds, fix findings in the working tree (uncommitted), run Phase 0–5 again, then commit one clean bundle — never push incremental fix commits.

> [!CAUTION]
> **Known failure modes (from real incidents):**
> 1. **Commit-SHA-blind polling** — matching a stale review against a previous commit and calling it "clean"
> 2. **Absence treated as clean** — treating bot silence as approval
> 3. **Incremental-commit push storm** — pushing every fix individually, turning one review into five
> 4. **Manual trigger on top of an active auto-review** — doubles the review volume for zero benefit. (Different from the auto-trigger fallback below: that fires ONCE after silence proves auto-review didn't fire — net cost is one review, not two.)
> 5. **Merge gate without SHA check** — merging when the "clean" review was against a different commit than HEAD
> 6. **Calendar-time bleed waiting on a silent auto-review** — auto-review can be silent on subsequent pushes. Without the auto-trigger fallback, the polling loop runs indefinitely. See §"Wait for the auto-review" below for the fallback.

```bash
BOT_LOGIN='<bot-login>'                 # your PR-review bot's GitHub login
PR_NUMBER=$(gh pr view --json number -q '.number')
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
HEAD_SHA=$(git rev-parse HEAD)
```

#### Wait for the auto-review

After pushing (whether the PR-opening push or a bundled follow-up), the bot typically posts within 60–180s. Poll at 60s intervals looking for a **review submission** (the `/reviews` API, not `/comments`) anchored to `HEAD_SHA` and submitted AFTER the push timestamp:

```bash
PUSH_TS=$(git log -1 --format='%aI' HEAD)
while true; do
  sleep 60
  REVIEW_FOUND=$(gh api repos/$REPO/pulls/$PR_NUMBER/reviews 2>/dev/null \
    | jq --arg bot "$BOT_LOGIN" --arg sha "$HEAD_SHA" --arg ts "$PUSH_TS" \
        '[.[] | select(.user.login==$bot) | select(.commit_id==$sha) | select(.submitted_at > $ts)] | length')
  if [ "${REVIEW_FOUND:-0}" -gt "0" ]; then break; fi
  # Auto-trigger fallback (see below) — fires once after CI passes + silence threshold
done
```

**SHA-PIN check:** every comment the bot posts carries a `commit_id`. For stop-condition purposes, only **review submissions** (the `/reviews` endpoint, NOT `/comments`) with `commit_id == HEAD_SHA` AND `submitted_at > PUSH_TS` count as the current review. The `/comments` endpoint lists only inline review comments — a "no findings" review submission has zero inline comments and won't appear there. GitHub also re-anchors older comments to newer SHAs when lines shift, which can falsely satisfy a `/comments`-only check; the `/reviews` + `submitted_at` check avoids both pitfalls.

> [!IMPORTANT]
> **Auto-trigger fallback:** bot auto-review IS reliable on the PR-opening push, but in some repos goes **silent on subsequent pushes** even when CI passes. Observed pattern: round-2 silent for hours after a fix push despite both CI checks SUCCESS; a single manual trigger comment fired the review within minutes.
>
> **Rule:** if both CI checks are SUCCESS on `HEAD_SHA` AND no bot review has landed within **5 minutes** of the latest CI completion, post the bot's manual trigger comment exactly ONCE, then continue polling. Do NOT post twice (would double-trigger if auto-review eventually fires). Implementation (replace `<bot-trigger>` with your bot's trigger string, e.g. `/gemini review`):
>
> ```bash
> CI_GREEN_AT=$(gh pr view $PR_NUMBER --json statusCheckRollup --jq \
>   '[.statusCheckRollup[] | select(.conclusion=="SUCCESS")] | (max_by(.completedAt) // .[0]).completedAt')
> # ... inside the polling loop, after the REVIEW_FOUND check:
> if [ -z "$AUTO_TRIGGERED" ] \
>    && [ -n "$CI_GREEN_AT" ] \
>    && [ "$(date -u +%s)" -gt "$(date -u -d "$CI_GREEN_AT + 5 minutes" +%s 2>/dev/null || gdate -u -d "$CI_GREEN_AT + 5 minutes" +%s)" ]; then
>   gh pr comment $PR_NUMBER --body "<bot-trigger>"
>   AUTO_TRIGGERED=1
> fi
> ```
>
> The "do NOT manually trigger" guidance in the §Anti-Patterns table refers to **double-triggering** (posting a trigger while auto-review IS firing) — that doubles review volume + burns quota. The fallback above triggers ONCE, only after silence proves auto-review didn't fire, so it adds zero quota cost vs. waiting indefinitely.

**If still no response 10 minutes after the auto-trigger fired:** the bot may be quota-exhausted or rate-limited. Document it in the report, do NOT treat silence as clean, surface to the user with the polling history.

#### Per-finding decision heuristic

Every bot finding gets classified with this table. This is the load-bearing discipline that keeps rounds finite:

| Classification | Criteria | Action |
|---|---|---|
| 🔴 **Critical** | Data loss, crash, security vulnerability, regression of a prior fix, sacred-rule violation | Fix this round — do NOT exit until resolved |
| 🟡 **Non-critical valid** | Fragility, style, equivalent-behavior refactor, missing test, polish/aesthetic suggestions | **Default: dismiss to BACKLOG.** Only fix if the change is trivially small (≤5 lines, one-file, no new abstractions) AND the fix genuinely improves correctness or UX. Avoid bot-spin: don't push a new commit for every cosmetic suggestion — each push kicks another auto-review and extends the cycle. When in doubt, backlog with `"Logged to BACKLOG — scoped to follow-up PR to bound blast radius"`. |
| ❌ **Invalid** | False positive: line-isolation read, misreads adjacent code, alleges compile error the analyzer doesn't see, alleges behavior the diff rules out | Dismiss with concrete rationale. Do NOT touch code. |
| 🔄 **Re-flagged** | Same finding the bot already posted on a prior SHA | Reply `"See #NNN dismissed with rationale: <link>"`. Do NOT re-debate. |

**Append every 🔴 critical AND every ❌ dismissal to your known-patterns file** with the canonical fix / rationale so future pre-push grep catches it or you can paste the dismissal rationale instantly next time.

#### Stop conditions (ALL must be true to exit Phase 6)

1. **The bot has reviewed the most recent code-bearing SHA** — there is at least one bot comment with `commit_id == LAST_CODE_SHA` AND `created_at` after that push. (HEAD may be ahead of LAST_CODE_SHA if doc-only commits followed — see doc-only escape below.) If not, keep polling.
2. **Every 🔴 critical finding is fixed in a pushed commit** OR dismissed with a rationale you can defend against a skeptical reader. No "I think this is probably fine" — make the case explicit.
3. **Every 🟡 non-critical finding has a reply** (fix SHA or backlog-with-rationale). Default to backlog unless the fix is trivially small AND materially improves correctness/UX — each push kicks another auto-review, so resist the urge to re-spin for polish.
4. **Every historical bot comment across the PR's entire life has a reply** (see Step 6.5).

When all four are true, Phase 6 is done — regardless of how many rounds it took.

#### Doc-only escape

The bot reviews code; doc-only pushes ride on the prior code-SHA's bot verdict. **Do NOT poll, wait for, or manually trigger the bot on doc-only pushes.** Each doc-only round-trip burns calendar time + bot quota for zero risk-management signal.

A push is "doc-only" iff every changed file matches one of:
- Under `docs/`
- Ends in `.md`
- Under the bot's config directory or `inputs/`
- Pure-comment changes inside source files (zero non-comment-line delta)

Verify with: `git show --name-only <sha> | grep -vE '^(docs/|.*\.md$|inputs/)'` (add your bot's config dir to the pattern) — empty output ⇒ doc-only.

For stop-condition #1 above: compute `LAST_CODE_SHA` as the most recent commit (from `main..HEAD`) that is NOT doc-only. If `HEAD == LAST_CODE_SHA`, the strict check applies. Otherwise the gate is satisfied when the bot has reviewed `LAST_CODE_SHA` (the doc commits after that pass through automatically).

Backstop pattern: bundle BACKLOG / handoff / spec drift updates INTO the same code-fix commit so there's no doc-only commit at all.

#### Convergence heuristic (round 3+ check)

Count 🔴 criticals in each round:
- Round N+1 should have FEWER criticals than round N
- If round N+1 has MORE criticals than round N, your fixes introduced bugs. Slow down, re-read the diff, consider reverting and starting the round over
- If the same finding is flagged 3+ times despite dismissals, paste the prior dismissal URL — do NOT re-debate

#### Bot escalation-loop detection

If the bot returns a finding on the SAME file:line for 3+ consecutive rounds (each round finding a NEW issue on the same artifact), STOP applying incremental fixes. Restructure to REMOVE the concrete artifact and defer its definition to the implementing phase.

**Canonical example:** four rounds on a single JSONL example in a spec — R1 type mismatch → R2 nested structure → R3 non-interface field → (R4 averted by restructure). Resolved by removing the example entirely and stating the contract in prose, deferring concrete format to the implementation phase.

**Concrete-artifact magnets in specs** (consider deferring rather than authoring):
- JSONL / JSON examples mirroring typed interfaces
- Function signatures cited in prose (must match exactly across all citations within the same file)
- Branch behind/ahead counts (drift as main advances; cite once + reference, don't repeat)
- Line-range citations (drift with source edits)

#### Round 3+ visibility ping

Before applying fixes in round 3 or later, post a one-line status message so the human has context (no decision requested, just transparency):

```
"Bot round 3 on PR #N — still addressing: <one-line summary of remaining critical>. Applying fix, will bundle and push. Prior rounds: <count, count, ...>."
```

#### Fix-and-bundle discipline (the anti-loop rule)

When the bot returns findings that need fixes:

1. **Apply every fix to the working tree** — do NOT commit or push yet
2. **Re-run Phase 3 hardening rules** on the working tree: fan-out grep for the same pattern, wrapper property audit, defensive-parsing sweep, strict analyzer
3. **Re-run Phase 5 cross-model review** (CLI subagent) on the working tree so a different model gets a pass before the bot does
4. **Commit everything as one bundle**: `git commit -am "fix: address bot round N findings + harden related patterns"`
5. **Push** — this triggers the next bot auto-review
6. **Loop** back to "Wait for the auto-review" above

Incremental fix commits are the single biggest cause of bot round-count explosion. One bot auto-review per push is fine; ten auto-reviews because you pushed ten fix commits is the failure mode this discipline prevents.

### Step 6.5 — History Audit (MANDATORY before declaring clean)

After receiving a clean bot response on the current HEAD, you MUST:

1. Fetch ALL bot comments across the entire PR history:
   ```bash
   gh api repos/$REPO/issues/$PR_NUMBER/comments --paginate | jq --arg b "$BOT_LOGIN" '[.[] | select(.user.login==$b)]'
   gh api repos/$REPO/pulls/$PR_NUMBER/comments --paginate | jq --arg b "$BOT_LOGIN" '[.[] | select(.user.login==$b)]'
   ```
2. For EVERY historical bot comment (from any round, any HEAD):
   - Verify it was either: (a) fixed with a committed change and a reply with the fix SHA, or (b) explicitly dismissed with rationale posted as a PR reply.
   - If ANY historical comment is unaddressed → fix OR dismiss it now, bundle the fix into the next commit, push. That push triggers the next bot auto-review.
3. **SHA-VERIFIED EXIT GATE** — Phase 6 exits cleanly only when ALL of the following are true:
   - The bot has reviewed the most recent **code-bearing SHA** (see doc-only escape in §Stop conditions above)
   - Zero open 🔴 critical findings on HEAD (per the Phase 6.4 decision heuristic)
   - Every 🟡 non-critical has a reply (fix SHA or backlog-with-rationale)
   - Every historical bot comment across all rounds has a reply
   - No code-bearing pushes have occurred since the last bot auto-review
   ```bash
   # Stop-condition probe before declaring Phase 6 done.
   # LAST_CODE_SHA is the most recent commit (from main..HEAD) that touched
   # any non-doc file. Doc-only commits (docs/, *.md, inputs/, bot config dir)
   # ride on the prior code-SHA's bot verdict.
   HEAD_SHA=$(git rev-parse HEAD)
   LAST_CODE_SHA=$(git log --pretty=format:'%H' main..HEAD | while read sha; do
     if git show --name-only --format='' "$sha" 2>/dev/null \
        | grep -qvE '^(docs/|.*\.md$|inputs/)'; then
       echo "$sha"; break
     fi
   done | head -1)
   LAST_CODE_SHA="${LAST_CODE_SHA:-$HEAD_SHA}"

   LATEST_BOT_SHA=$(gh api repos/$REPO/pulls/$PR_NUMBER/comments --paginate 2>/dev/null \
     | jq -r --arg b "$BOT_LOGIN" '[.[] | select(.user.login==$b)] | sort_by(.created_at) | last.commit_id // ""')

   if [ "$LATEST_BOT_SHA" != "$LAST_CODE_SHA" ] && [ "$LATEST_BOT_SHA" != "$HEAD_SHA" ]; then
     echo "🔴 The bot has not reviewed the most recent code-bearing SHA ($LAST_CODE_SHA). Wait or push a bundled fix."
     exit 1
   fi
   echo "✅ Bot gate: HEAD=$HEAD_SHA last-code=$LAST_CODE_SHA last-bot=$LATEST_BOT_SHA"
   ```

---

## Phase 7: REPORT

**Goal:** Summarize everything. Append audit log. Hand off to staging release for compliance + build verification (it stops at an approved, unmerged PR); production release then merges and distributes.

> [!CAUTION]
> **MANDATORY PRECONDITION — DO NOT SKIP**
> Before writing ANY part of this report, you MUST verify ALL of the following are true:
> 1. You executed Phase 6 (Push, PR & Bot Gate) — not just Phase 5 (cross-model convergence)
> 2. You have a specific commit SHA from an explicit bot response (not from your own analysis)
> 3. That SHA matches `git rev-parse HEAD` (you verified this in Step 6.5)
> 4. The response was from the PR-review bot on GitHub, NOT from a CLI subagent or your own review
>
> **If ANY of these are false, you CANNOT write this report.** Go back to the phase you skipped.
> **If you did not run Phase 6 at all** (e.g., session ended, context limits), report status as:
> `"🔄 ANOTHER ROUND NEEDED — bot gate not yet executed. Cross-model convergence passed but the bot has not reviewed the PR."`

### Convergence Assessment (MANDATORY)

```markdown
## Convergence Assessment
- Findings fixed (total across all rounds): {count}
- Findings dismissed with rationale: {count}
- Cross-model rounds completed: {count}
- Bot rounds completed (auto-reviews): {count} ← MUST be ≥1. If 0, the bot hasn't seen HEAD yet.
- Bot last-review SHA: {sha} ← the commit_id of the most recent bot comment
- Bot SHA matches HEAD: {yes/no} ← MUST be "yes" (either findings on HEAD, or no findings anchored to a newer push)
- Open 🔴 critical findings on HEAD: {count} ← MUST be 0 to declare converged
- Open 🟡 non-critical findings on HEAD: {count with backlog refs}
- Convergence trajectory: {round-by-round critical counts, e.g. "5 → 2 → 0"} ← should trend down
- Structural changes introduced: {yes/no}
- Confidence level: {HIGH / MEDIUM / LOW}
- PR: #{number}
- **Status:** {One of:}
  - "✅ CONVERGED — PR #{number}: bot reviewed HEAD {sha}; 0 open criticals; all 🟡 dispositioned. Ready for staging release."
  - "🔄 ANOTHER ROUND — {open-critical-count} criticals remaining on HEAD. Applying fixes in working tree, will bundle and push."
  - "⚠️ BOT UNRESPONSIVE — Review not received after 10 min on HEAD {sha}. Documented as exceptional; proceeding requires explicit user override."
  - "🛑 ESCALATE — {what's stuck and why}"
```

### Audit Log

If ANY code was modified, append to `docs/peer-review-log.md`:

```markdown
## Peer Review — {date}
**Branch:** {name} | **Reviewer:** {model} | **Builder:** {model from handoff}
**Business Goal:** {one-line}
**Cross-model rounds:** {N} | **Bot rounds:** {N}
**Bot-clean SHA:** {sha}

| # | Severity | File | Description | Resolution |
|---|----------|------|-------------|------------|

### Verification: {test/build/lint results}
### QA Cases Added: {count} new cases
### Rollback: `git revert HEAD~{N}`
```

### Report to Conversation

Deliver a conversation artifact covering: context, safety property results, findings summary with resolutions, verification results, QA cases added, goal assessment, convergence assessment, bot gate results, and deferred items.

### Next Step

> [!CAUTION]
> **FINAL ANTI-HALLUCINATION CHECK** — Before delivering this message, answer these questions:
> 1. Did the PR-review bot post a comment on the PR? (Not a CLI subagent, not your own analysis)
> 2. What is the exact SHA in that comment? Does it match `git rev-parse HEAD`?
> 3. Did you run `gh api` to fetch and parse the bot's actual response?
> If the answer to ANY of these is "no" or "I'm not sure", you MUST NOT deliver the message below.
> Instead say: "Cross-model review converged but I have not yet verified a clean bot report. Run `/peer-review` again to execute the bot gate."

Tell the human: "Peer review complete — cross-model converged and bot-clean on PR #{number} (SHA: {sha}). Run the staging-release workflow for compliance checks + build verification (it stops at an approved, unmerged PR); then the production-release workflow merges and distributes." (See [Staged Release](../../workflows/release-prep.md).)

---

## Edge Cases

| Scenario | Action |
|----------|--------|
| **No state/handoff doc** | ASK user: what was the goal, what changed, what does "done" look like? |
| **No tests exist** | 🟡 finding. Build + lint as verification. Add test cases to QA_CASES.md. |
| **Multi-repo changes** | Review current repo only. Note cross-repo dependencies as findings. |
| **Code correct but wrong goal** | 🔴 finding. Flag gap between implementation and business goal. |
| **Sacred rule violation** | Always 🔴. No exceptions, no downgrades. |
| **CLI subagent unavailable** | Self-review only. Note in report that cross-model review was not possible. Recommend the human switch models manually. |
| **No open PRs (Phase -2)** | Skip directly to Phase -1. |
| **Bot debt on stale/abandoned PRs** | Only fix PRs whose branch is actively being worked on (matches current branch or state-doc context). Report stale PRs to the human for manual triage. |
| **Dirty working tree at Phase -2 start** | Stash uncommitted changes before switching branches. Restore after returning to original branch (Step -2.5). |
| **No unfinished items (Phase -1)** | Skip directly to Phase 0. |
| **PR already exists for this branch** | Use existing PR. Do NOT create a duplicate. |
| **Bot finds issues requiring code changes** | Classify each finding per the Phase 6.4 decision table. Fix criticals in the working tree → run Phase 3 pre-push hardening rules → run Phase 5 cross-model review → commit everything as one bundle → push. Bundle discipline is non-negotiable. |
| **Bot unresponsive > 10 min** | Exceptional. Document the quota-out / rate-limit state in the report. Do NOT silently treat silence as approval. Only proceed with explicit user override. |
| **Same finding re-flagged across multiple SHAs** | Paste the prior dismissal URL as the reply. Do NOT re-debate or re-rationalize. Append to your known-patterns file under "known false positives" so the next run catches it pre-push. |
| **Convergence heuristic fails** (round N+1 has more criticals than round N) | Your fixes introduced regressions. Slow down, re-read the bundled diff, consider reverting the last commit and starting the round over with tighter scope. |
| **Bug found after merge to `main`** | STOP. Do NOT fix on `main`. Run `git checkout -b fix/<description>` immediately. Fix on the branch, then re-enter `/peer-review` from Phase 0. The full pipeline runs again — no shortcuts for "it's just a small fix." |
| **Currently on `main` or `release`** | Before ANY code edit, checkout a feature branch. If you have already made uncommitted changes on `main`, stash them (`git stash`), create a branch, then `git stash pop`. |

---

## Anti-Patterns

| Don't | Do Instead |
|-------|------------|
| Flag hypothetical issues without concrete scenarios | Apply the anti-hallucination gate — cite line, describe failure |
| Refactor outside the blast radius | Log to BACKLOG, don't touch |
| Iterate >3 times on the same failure | Stop, report stuck state, escalate |
| Skip the bot's sacred-rules file | Always read repo instructions first |
| Trust LLM judgment over test output | Run the tests, trust the exit code |
| Apply a subagent's suggestions blindly | Verify each independently before applying |
| Declare convergence after fixing bugs | Fixing bugs resets the clean-pass counter |
| Rush through QA case additions | Every review must leave the QA suite stronger |
| Push to `main` or `release` | Feature branches only. NEVER push to protected branches. NEVER merge. |
| Manually trigger the bot ON TOP OF an active auto-review | The bot auto-reviews every push. A manual trigger on top of an in-flight auto-review just burns quota. **EXCEPTION:** the auto-trigger fallback in §"Wait for the auto-review" — fire the trigger once after CI passes + 5min of silence — IS REQUIRED when your bot goes silent on follow-up pushes. |
| Push incremental fix commits | Every push triggers another bot auto-review. Bundle fixes into one commit: fix in working tree → Phase 0–5 hardening → single commit → push. |
| Exit Phase 6 with unresolved critical findings | A 🔴 critical is fixed-and-pushed OR defensibly dismissed with a written rationale. Anything else is an abort. |
| Skip the bot history audit | A "clean" latest round means nothing if prior findings are unresolved. Every historical comment needs a reply. |
| Treat bot silence as clean | "No response" is not "approval." Document as exceptional and only proceed with explicit user override. |
| Dismiss findings without rationale | Every dismissal needs a concrete reason a skeptical reader would accept ("line 64 emits failure state at lines 66-76", not "I don't think this is a problem"). |
| Re-debate re-flagged findings | If the bot flags the same pattern a 3rd time, paste the prior dismissal URL. Do NOT write a new rationale. |
| Rush to PR before reviews converge | All internal hardening (Phases 0-5) must complete before the first push. Every issue caught locally saves a bot round. |
| Fix bugs directly on `main` after a merge | ALWAYS branch first (`fix/<desc>`), run full `/peer-review` pipeline. "It's just one line" is how production incidents start. |
| Claim "the bot reports zero findings" after cross-model convergence | Cross-model convergence (Phase 5) ≠ bot clean (Phase 6). The bot is an EXTERNAL service on GitHub. You have not interacted with it until Phase 6. Saying "bot clean" before Phase 6 runs is a process hallucination. |
| Equate CLI subagent review output with the bot's output | The CLI subagent runs locally. The bot runs on GitHub PRs. They are different systems with different review criteria. |
