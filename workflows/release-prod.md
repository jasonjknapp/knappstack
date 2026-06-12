---
description: Production release workflow. Run after the staging gate has passed and a human has approved the review-clean PR. This is where the merge happens and the build is distributed.
---

# Release Prod Workflow (`/release-prod`)

This is the executable runbook for the production stage — the merge → build → distribution step of the staged release path. Install per [SETUP.md](../SETUP.md) and wire each platform step to your own CI, hosting target, error tracker, and backup mechanism.

**Purpose:** Merge the approved PR, distribute the build to production, verify it works, and clean up.
**Entry requirement:** A passed staging gate (`/release-prep`) with explicit human go-ahead. The PR is **review-clean and UNMERGED** — performing the merge + production distribution is THIS workflow's job (the staging workflow never distributes).
**Exit state:** Code merged + distributed, verified, documented, ledgers updated.

> **Do not rush.** Verify everything. Back up everything. Check every account.

> [!IMPORTANT]
> **Review model:** the authoritative review protocol lives in [Cross-Model Adversarial Review](../skills/peer-review/SKILL.md). The staging workflow already ran the automated PR-review gate — `/release-prod` should NOT re-trigger it. If a regression or blocker surfaces after staging approval, that's a review reset (back to the hardening loop), not a fresh standalone gate round.

> [!IMPORTANT]
> **Doc-only commits between staging-approval and prod-deploy do NOT need a fresh review gate.** Pushes that touch only docs, `*.md`, your bot's config, or staging scratch files — or are pure-comment edits — ride on the prior code-SHA's review verdict. The Phase 0 staleness check ("verify no new commits on feature branch since QA") still applies to **code-bearing** commits — doc-only commits pass through. If unsure whether a commit is doc-only, run `git show --name-only <sha>` and check whether anything outside the doc/config set changed; if not, the commit is doc-only.

---

## Phase 0: Entry Gate (HARD — Cannot Skip)

1. **Verify the staging workflow completed:**
   - Ask for the PR number or look up the latest entry in your releases-in-flight ledger.
   - Verify status is `approved` (not just `staging-deployed` / `build-verified`).
   - If status is NOT approved → STOP: "Run the staging workflow first and get a human go-ahead."
   - **Verify the PR is still OPEN / unmerged** (`gh pr view <#> --json state,mergedAt` — `mergedAt: null` ⇒ unmerged; note `merged` is NOT a valid `--json` field). The merge is THIS workflow's job. If it's already merged, the merge already happened — investigate before proceeding.

2. **Staleness check:**
   - Check when the PR was approved (the `Approved:` timestamp in the ledger).
   - If >24 hours ago:
     - Run `git log main --oneline --since="<approval-timestamp>"` to check for new commits on the trunk.
     - If new commits exist → re-run the automated staging regression from the staging workflow.
     - If clean → proceed.

3. **Verify no new commits on feature branch since QA:**
   ```bash
   git log origin/<branch>..HEAD --oneline
   ```
   If any unreviewed commits → STOP: "New commits since approval. Re-run the automated staging regression."

---

## Phase 1: Account & Environment Lock

4. **Load context and verify identity:**
   ```bash
   pwd
   git remote -v
   git config user.email
   git branch --show-current
   ```

5. **Cross-reference against your project/account registry** (the map of which repo deploys to which account/team/project — a wrong-account deploy is a category of disaster):

   | Check | Command | Must Match |
   |-------|---------|-----------|
   | Remote org | `git remote -v` | Registry entry for this repo |
   | Author email | `git config user.email` | Registry entry |
   | Hosting team/project | `cat <platform config> 2>/dev/null` | Correct team/project for this repo |
   | Datastore project | `cat <datastore config> 2>/dev/null` | Correct project ID |
   | Clean tree | `git status --short` | Must be empty |
   | Branch | `git branch --show-current` | Must be on `main` or the feature branch (NOT `release`) |

6. **Repo allow-list HARD BLOCK:**
   - If the remote org is not on your allowed-to-deploy list → ABORT: "Cannot deploy this repo."

7. **Build mode safety:**
   - If the repo uses `.env` files: verify the production build will use `.env` (NOT `.env.staging`, `.env.dev`).
   - NEVER build with one environment's config and then deploy to another target (bakes in the wrong auth domain / API base).
   - For all: verify the build command matches the deploy target.

8. **Verify active CLI sessions (if a CLI deploy is needed):**
   ```bash
   # Only run the ones this repo actually uses:
   <hosting-cli> whoami 2>/dev/null
   <datastore-cli> projects:list 2>/dev/null | head -5
   ```
   Verify the logged-in account matches the registry.

---

## Phase 2: Backup Everything

> **No backup = no deploy. Non-negotiable.**

9. **Git branch backup:**
   ```bash
   BACKUP_NAME="backup/$(date +%Y-%m-%d)-pre-$(git branch --show-current | sed 's/feature\///')"
   git branch $BACKUP_NAME origin/release 2>/dev/null || git branch $BACKUP_NAME origin/main
   git push origin $BACKUP_NAME
   ```
   - Verify the backup was pushed:
   ```bash
   git branch -r | grep "$BACKUP_NAME"
   ```

10. **Data backup (if the release touches writes):**
    - Check the write-path classification from the staging workflow.
    - If anything other than "No writes," snapshot/export the affected data store *before* the deploy, using your datastore's export mechanism:
    ```bash
    <datastore-export> <destination>/pre-release/$(date +%Y-%m-%d)-<branch-name>
    ```
    - **Verify the backup completed** (list the destination; confirm the export's completion marker is present).
    - If verification fails → STOP: "Data backup did not complete. Cannot proceed."

11. **Log backup status and proceed immediately:**
    > "Backups complete:
    > - Git: `backup/<name>` pushed to origin ✅
    > - Data: `<destination>/pre-release/<name>` verified ✅ (or N/A)
    > Proceeding to merge and deploy."

---

## Phase 3: Merge & Deploy

12. **Merge to the production branch (platform-specific):**

    **Web repos with a `release` branch:**
    ```bash
    git checkout release
    git pull origin release
    git merge main
    git push origin release
    ```
    Your host auto-deploys on push to `release` (or run the explicit deploy in step 13).

    **Mobile app (app-store build) — this is where store distribution happens:**
    - The version bump rode in the staging PR. **Re-verify the store/build inventory FIRST** — if another build shipped while the PR sat unmerged, bump the build number on the branch and push before merging so the number is fresh and the store won't silently drop a duplicate.
    - **Squash-merge the approved PR to `main` NOW** if your CI is wired to build `main` and upload to the store on merge:
    ```bash
    gh pr merge <#> --squash --delete-branch
    ```
    - **HUMAN-GATED RELEASE NOTE:** if your CI does NOT auto-distribute on merge (the common, safer default), the actual store upload is a deliberate human action — STOP and ask the human to trigger the production distribution workflow in your CI on the `main` branch. Mobile store submission is a one-way door; keep a human on the trigger.
    - **DO NOT merge `main` into `release` yet.** The mobile `release` branch must always reflect the version currently live in the store (for production debugging). The merge to `release` happens in Phase 5 (step 24a) only after the human confirms the build is live.

    > [!IMPORTANT]
    > **Mobile `release` branch rule:** `release` = what's in the store right now. Never update it until the new version is actually live. This lets you `git diff release..main` to see exactly what changed vs production at any point during QA.

    **Backend (server functions — manual deploy):**
    ```bash
    <function-deploy> --only <specific-function-name>
    ```
    > ⚠️ NEVER deploy the whole function set blind. ALWAYS specify the function name.

13. **Run the platform-specific production deploy (if not auto-deployed):**

    | Target | Production Deploy |
    |------|-------------------------|
    | Web host (with `release` branch) | Auto (host deploys on push to `release`) |
    | Static / CDN site | Explicit deploy command to the production hosting target |
    | Backend functions | `<function-deploy> --only <specific-function-name>` |
    | Mobile app | **Human-gated** (trigger production distribution in CI; see step 12) |

---

## Phase 4: Production Verification

### 4a. Immediate Verification (within 5 minutes)

14. **Web properties (drive a real browser):**
    - Load the production URL.
    - Verify the page renders, no console errors, content is correct.
    - Click through 2-3 core flows.

15. **CDN / asset verification:**
    ```bash
    curl -sI https://<production-asset-url> | head -5
    ```
    - Must return `200 OK`. A successful deploy does NOT guarantee the file actually uploaded — verify the artifact, not just the deploy exit code.

16. **Rendered-site verification:**
    - Drive a browser: load homepage and 1-2 key detail pages.
    - Fetch the homepage HTML and verify critical tags (SEO/meta) are present.

17. **Backend function verification:**
    - `curl` the HTTP function endpoints and verify expected response codes.
    - Check the logs for startup errors:
    ```bash
    <function-logs> <function-name> --limit=20
    ```

18. **Mobile — print the manual checklist:**
    ```
    📱 Mobile QA Required (manual):
    - [ ] Install the latest build from the store/beta channel
    - [ ] App launches without crash
    - [ ] Core feed/list loads with media
    - [ ] Detail screen renders correctly
    - [ ] Auth / create-account flow works (if auth touched)
    - [ ] Primary create/submit flow works
    - [ ] Push notifications received (if notifications touched)
    - [ ] Any gated flow triggers correctly (if onboarding touched)
    ```

### 4b. Schema-Change Regression (MANDATORY if schema touched)

19. Even though staging passed, **re-run on production:**
    - Load the production pages/widgets that consume the changed schema.
    - Verify all dependent fields populate (names, media, links, full data display).

### 4c. Error-Tracker Soak (15-minute window)

20. **Capture baseline:**
    - Fetch the current error count from your error tracker (e.g. Sentry).
    - Note the count and timestamp.

21. **Wait 15 minutes.**

22. **Re-check the error tracker:**
    - If new errors appeared post-deploy that weren't present before → flag immediately.
    - If the error rate spiked → notify the human: "New errors detected after deploy: [summary]. Investigate or rollback?"

### 4d. Canary Verification (Backend functions only)

23. If a backend function was deployed that serves real client traffic:
    - Pick ONE live client/hostname.
    - Drive a browser: load the page that exercises the function.
    - Verify it loads and renders the expected data.
    - Only after this verification, declare the function deploy successful.

---

## Phase 5: Ledger Cleanup & Documentation

24. **Update the releases-in-flight ledger:**
    - Move the entry to the **Completed** section with the deploy date.

    24a. **Mobile only — Trailing `release` merge:**
    - If the human has NOT confirmed the app is fully approved and live in the store:
      - **DO NOT run this step.**
      - Output a reminder: "Your app is now in store/beta QA. After you submit it and it goes live, invoke me again and say 'Mobile is live, update release branch' and I'll run the merge."
    - If the human HAS confirmed the app is live:
      ```bash
      git checkout release
      git pull origin release
      git merge main --no-edit
      git push origin release
      ```

25. **Update your work/business ledger:**
    - Mark deployed items complete.

26. **Session-state save — invoke `/checkpoint`** (MANDATORY):

    Run your canonical checkpoint/session-state save so the just-completed deploy lands in the right places. This is the comprehensive save (preferred over an ad-hoc subagent doc update, which produces narrower, less-anchored results).

    What the checkpoint covers (so you don't duplicate this work):
    - **Session-state handoff** — MUST cite this deploy's SHAs, timestamps, the anti-deletion delta, the soak result, and any verbatim user-goal quotes from the release conversation.
    - **Plan/spec files** — bump status + add a phase-status note citing the merge SHA + PR number + deploy date if a phase shipped. Cross-link any sibling-repo PRs.
    - **Review log** — one entry per PR shipped this session: cross-model rounds + automated-gate rounds + cross-repo PRs + notable findings.
    - **Backlog** — mark resolved items, log post-deploy follow-ups, downgrade items now subsumed by what shipped.
    - **Plan index** — record any spec-status transitions and bump `last_updated`.
    - **Plan-state validator** — must pass before commit (for repos using durable plan-state headers; see [Durable Planning](../skills/plan/SKILL.md)).
    - **Cross-repo alignment** — if the backend touched fields visible to another surface, surface it in that repo's handoff; if mobile shipped the same release, cross-link SHAs.
    - **Workflow sync** — version-control any agent-config / command edits made during this release. **This subsumes step 30 below.**

    **Spec archive sub-step:** if a spec transitioned to *all-phases-complete* (not just a phase shipped), archive it during the checkpoint pass and update the plan index + cross-references in the same commit.

    **Anti-drift discipline (CRITICAL):** before overwriting any handoff narrative or spec content, REFRESH MEMORY by re-reading the canonical doc you're updating. Conversations get compacted; summary memory drifts. The cost of one extra `Read` is far lower than the cost of writing a stale fact into a doc that future sessions cite as authoritative. (Generalizable lesson from a real post-compaction drift incident.)

27. **Review build/deploy logs for warnings:**
    - Errors → must fix immediately.
    - Deprecation warnings → log to the backlog as tech debt.
    - Build warnings → evaluate severity, log significant ones.

28. **Issue-tracker cleanup:**
    - Move shipped tickets to "Done."

29. **Stale Branch & PR Cleanup (MANDATORY):**
    After every successful production release, close out all superseded work:
    ```bash
    # List all open PRs — identify any that were superseded by this release
    gh pr list --state open
    ```
    For each open PR whose changes are now **fully superseded** by the code just shipped:
    a. Comment explaining why it's being closed:
       ```bash
       gh pr comment <NUMBER> --body "Closing: this work was superseded by the v<VERSION> release (PR #<MERGED_PR>). All relevant changes have been incorporated or intentionally replaced."
       ```
    b. Close the PR: `gh pr close <NUMBER>`
    c. Delete the remote branch: `git push origin --delete <branch-name>`
    d. Delete the local branch: `git branch -D <branch-name>`

    e. **Auto-prune merged branches.** After every successful merge to `main`, sweep accumulated dead branches so they never pile up:
       ```bash
       # (1) Local backup/* branches merged into main → delete locally.
       #     Remote copies (pushed in Phase 2 step 9) are RETAINED as the durable
       #     rollback net, so local deletion loses nothing. Keep THIS release's
       #     backup locally as the immediate rollback target until the next release.
       CURRENT_BACKUP="backup/$(date +%Y-%m-%d)-pre-<branch>"   # this release's (keep it)
       for b in $(git branch --merged main | grep 'backup/' | sed 's/^[* ]*//'); do
         [ "$b" = "$CURRENT_BACKUP" ] && continue
         git branch -d "$b"        # -d (safe): refuses if not actually merged
       done
       # (2) Other local branches git can prove are merged (true ancestors of main).
       git branch --merged main | grep -vE '^\*| main$|backup/' \
         | sed 's/^[ ]*//' | xargs -r -n1 git branch -d
       ```
       - Use `-d` (safe delete), never blanket `-D` — `-d` refuses any branch with unmerged commits, so this can't destroy WIP.
       - **Squash-merged stragglers** (old feature branches whose work shipped via squash, so `git --merged` can't see them and `-d` refuses them) accumulate separately. Do NOT blanket-`-D` them — surface the list to the human and `-D` only the ones confirmed dead (squash rewrites patch-ids, so `git cherry`/`--merged` can't auto-prove safety). This keeps the auto-prune zero-risk.

    > [!WARNING]
    > **MULTI-WORKSPACE SAFETY:** Do NOT close PRs that contain **unmerged, still-relevant work** (e.g., a feature branch for the next version), or ANY PR committed to in the last 24 hours. Assume recently active branches are active WIP in another agent window and **leave them alone**. Only close PRs whose changes are definitely and fully obsolete. When in doubt, ask the human. The step-e auto-prune is safe under this rule because `-d` only removes branches already fully in `main` (an active WIP branch by definition is not).

    Also clean up any temporary files left by agent sessions:
    ```bash
    # Check for scratch files in the repo root
    git status --short | grep -E '^\?\? .*(\.py|\.js|\.sh|test_|fix_|poll_|update_)'
    # If found, delete them (they should never be committed)
    ```

30. **Sync workflows** — SUBSUMED by the checkpoint's workflow-sync step (run as part of step 26 above). **If step 26 was somehow skipped** (it shouldn't be — it's MANDATORY), run your workflow-sync script directly. The canonical path is via step 26; if you find yourself running it standalone here, ask why step 26 was skipped.

31. **Final success summary:**
    Report to the human:
    - What shipped (features, fixes).
    - Production URLs verified.
    - Error-tracker status (clean or issues found).
    - Any follow-up items logged to the backlog.
    - Link to the completed releases-in-flight entry.

---

## Self-Improving QA Loop (MANDATORY — All Projects)

Every bug that reaches production — and every reviewer finding that should have been caught earlier — becomes a permanent QA case. The QA list only grows; it never silently shrinks. This is the shared loop with the staging workflow: a failure caught in prod is a missing test, and the fix isn't done until that test exists. See the `QA_CASES.md` template for the living format.

---

## Rollback Procedures

If something goes wrong after a production deploy — a clean rollback beats a hopeful fix-forward under pressure.

### Web host (auto-deploy on push)
1. In your host's dashboard → Deployments → find the last good build → **Promote to Production**.
2. Git: `git checkout release && git reset --hard backup/<last-good> && git push --force-with-lease origin release`

### Static / CDN site
1. In your host's console → release history → **Rollback** to the last good release.
2. Git: same as above.

### Backend functions
```bash
git checkout backup/<last-good>
<function-deploy> --only <specific-function-name>
```

### Data store
```bash
<datastore-import> <destination>/pre-release/<backup-name>
```
> ⚠️ Import OVERWRITES existing data for imported collections. Coordinate with the code rollback.

### Mobile (hotfix)
```bash
git checkout main && git reset --hard backup/<last-good>
# Bump version+build (MUST be higher than any previous upload)
git commit -am "hotfix: revert to stable build"
git push origin main --force-with-lease
# Then HUMAN-trigger the production distribution in CI → store review
```
