---
description: Mid-session workflow that aligns all documentation with the current AI session's work, captures learnings, and saves a cross-conversation handoff. Session-scoped only.
---

# Checkpoint Workflow

When the user types `/checkpoint` (or asks to save state / document the session), execute the following steps. Install per SETUP.md.

This is the **session-triggered half** of the self-improvement loop described in the lifecycle ([../concepts/ai-dev-lifecycle.md](../concepts/ai-dev-lifecycle.md), "Checkpointing"): a session ending → capture what was learned into durable state. The **bug-triggered half** lives in your living QA case list ([../templates/QA_CASES.md](../templates/QA_CASES.md)) and is driven by [/debug](../skills/debug/SKILL.md). Same goal — the system gets measurably better every cycle — two entry points. Give each kind of learning **one home**.

## Scope — what /checkpoint IS and IS NOT

`/checkpoint` does ONLY work that requires the current AI conversation's context:
- Capturing session decisions, discoveries, and state into the handoff doc
- Aligning plan/spec files with code shipped this session
- Logging cross-model + PR-review rounds for PRs shipped this session
- Recording knowledge gaps, feedback, and learnings from this session
- Committing the resulting doc updates

`/checkpoint` does NOT do work that operates on persistent state across sessions or at the system level. That kind of cross-workspace housekeeping — global ledger syncs, recurring inbox grooming, scheduled archiving — belongs to your start-of-day / end-of-day routines, which run ONCE at IDE/system level and have no individual AI session context. If you find yourself running a cross-workspace sync or rewriting a long-lived index during a checkpoint, stop — that's the wrong workflow.

## Mode — Full vs Compact

Checkpoint runs in one of two modes. Detect which applies before step 1:

- **Full mode (default)** — invoked manually via `/checkpoint`. Runs all steps 1–13.
- **Compact mode** — invoked automatically by a pre-compaction hook (or env var `CHECKPOINT_MODE=compact`) just before the agent auto-compacts its context. Runs steps 1–10, 12, 13. SKIPS step 11 (git commit) only. Rationale: pre-compaction fires mid-work; auto-committing would bundle half-done code changes with doc updates. Leave the working tree dirty; the post-compaction session commits when appropriate.

If unsure, assume Full mode.

## Repo convention detection (determines handoff format)

Check whether the repo uses the **durable-planning convention** ([../skills/plan/SKILL.md](../skills/plan/SKILL.md)). Signals:
- The handoff doc opens with a live-plans / routing block (possibly preceded by a `## 🛑 FRESH SESSION — READ FIRST` checklist)
- Plan files carry a `<!-- PLAN-STATE v1 -->` header
- `CLAUDE.md` has a retention / anti-paraphrase section

If the repo uses the durable-planning convention → follow the durable-planning variants in steps 2, 6, 8 below. If NOT → follow the legacy variants (explicitly labelled as such). Consider porting any repo that isn't already on the convention — it's what keeps decisions-in-prose from drifting across sessions ([../skills/plan/SKILL.md](../skills/plan/SKILL.md)).

## Parallel-Session / Cross-Thread Coordination

More than one agent session can be running on the same repo simultaneously (e.g. one on a payments feature, another on a data pipeline). Without discipline, they stomp on shared docs. Apply these rules during every checkpoint:

- **Branch ownership.** Each session stays on branches prefixed with its workstream. Never check out or commit on another session's branches. Never edit files in a sibling repo another session owns.
- **Doc ownership.** Each spec file and its matching plan file is owned by a single session. Only the owning session edits it. Cross-cutting docs are partitioned by workstream.
- **Shared-doc append zones.** The handoff, the backlog, the plan index, and the peer-review log are shared. For these:
  - Append only; never rewrite or delete items another session created. If something looks stale, tag `@needs-review` and flag in the handoff — don't delete.
  - Tag new items with `<!-- OWNED BY: <workstream> -->` or inline `@owner:<workstream>` so the other session can see who wrote what.
  - Before editing, run `git fetch && git log origin/main -- <file>` to check for recent sibling commits; rebase/pull if newer.
  - On conflict, preserve BOTH sessions' content; never pick one side without explicit user direction.
- **Cross-session discovery.** At the start of every checkpoint, run `git log --all --since='24 hours ago' --format='%h %ae %s' | head -20` to see what other sessions committed recently. If you spot another workstream active, read its last commit to understand what it's doing.
- **Handoff cross-references.** In your own session's handoff narrative, include a `## Parallel sessions` subsection naming the other active workstreams you know about, with each one's branch and last-seen note. This lets the next session instance orient faster.

---

## Steps

1. **Summarize the Session.** Briefly list the concrete value created and decisions made during the current working session. Capture session context that only the current AI conversation has — external commitments, deliverable language, deadlines promised, directives issued.

2. **Local Handoff + Backlog Update (session state).** Update the local workspace's handoff doc with full-fidelity session state — context, decisions, directives, and exact next steps. Also update the backlog if applicable: mark completed items, add new discoveries.
   - **Durable-planning repos (primary path):** update ONLY the session-state narrative section BELOW the live-plans / routing block's end marker. Do NOT edit the routing block unless a plan ACTUALLY changed phase state (proposed → in_progress → awaiting_review → complete, or a plan was added/archived). Do NOT rewrite the FRESH SESSION CHECKLIST at top — that's the cross-conversation recovery artifact. Do NOT create a parallel routing doc (no `CURRENT.md` / `CANONICAL.md` / `STATUS.md` — see [../skills/plan/SKILL.md](../skills/plan/SKILL.md) §2).
   - **Legacy repos:** update the handoff per the legacy template (see step 6).
   - **CRITICAL (all paths): Transfer deliverable language verbatim.** If you promised "I'll get you something today," the handoff must say exactly that — not "finalizing deliverables." Include deadlines, recipient names, commitment language, and amounts with zero abstraction.
   - **CRITICAL (all paths): Capture external commitments.** Any email sent, meeting scheduled, or deadline promised during this session MUST appear in the handoff (in "What's NOT DONE" for legacy repos, in the session-state narrative for durable-planning repos) with the exact date and recipient.
   - **CRITICAL (all paths): Capture the user's verbatim goal statements from this session.** If the user articulated a goal, constraint, or architectural decision in this session ("I want X", "it should do Y", "never Z"), copy it VERBATIM into the session-state narrative — NOT your paraphrase of it. Format: `**User goal (YYYY-MM-DD):** "<exact quote>"`. This becomes the anti-paraphrase anchor for any future session reviewing or extending the work. Drift almost always traces to a *summary* of a goal diverging from what the human actually said; verbatim quotes let future sessions cite, not paraphrase ([../skills/plan/SKILL.md](../skills/plan/SKILL.md) §3–§4).

3. **Persist Learnings (Knowledge Base).** If any new patterns, tools, or best practices were explored, update or create the appropriate files in your agent-config knowledge directory. These are insights from THIS session that future sessions need.

4. **Knowledge Gap Audit (Mandatory).** Review the session for facts that were re-discovered from scratch, claims that turned out wrong, or context that required multiple tool calls to establish. For each gap:
   - **Re-discovered knowledge** — a fact that already existed somewhere but wasn't found until mid-session. Fix: add it to the appropriate knowledge file with a clear header, or add a cross-reference from the file that *should* have surfaced it.
   - **Wrong claims** — any assertion made to the user or in a deliverable that turned out inaccurate (e.g. claiming something was "built" when it wasn't). Fix: log it to your feedback log with the root cause, and add ground-truth data to the relevant knowledge file so future sessions have the correct state.
   - **Missing verification patterns** — any claim that required manual verification (e.g. curling a URL to check whether a script is deployed). Fix: add the verification command as a runbook entry in the relevant knowledge file.
   - **Stale documentation** — any knowledge file read that contained outdated information. Fix: update it with current state and a timestamp.
   - Update relevant skill/command files with new rules or banned patterns.
   - Add or revise workflow steps that were missing or caused errors.
   - **For a bug/incident specifically**, the hardening loop and the "where learnings land" destination rule live in the QA-case list ([../templates/QA_CASES.md](../templates/QA_CASES.md)) and the staged-release flow ([/release-prep](release-prep.md)). This audit is its session-level sibling: capture session learnings here, route bug→test hardening there.
   - Goal: every checkpoint leaves the system measurably smarter. If zero gaps are found, state that explicitly.

5. **Feedback Log.** Append any corrective feedback from this session to your feedback log. Include the corrective moment verbatim + why the original approach was wrong + the rule going forward.

6. **Handoff (Cross-Conversation State Save).** For every project/domain actively worked on this session, **read the existing handoff doc first**, then update it:
   - **Merge, don't overwrite.** Preserve still-relevant context from previous sessions. Only remove items that are fully resolved.
   - **Resolved items** → move to permanent storage: completed work to the backlog's Completed section, learnings to the knowledge directory, patterns to the relevant skill/workflow file. Don't just delete — archive to the right place.

   **Durable-planning repos (primary path):**

   The handoff is already structured for cross-conversation recovery — the live-plans / routing block and optional `## 🛑 FRESH SESSION — READ FIRST` checklist ARE the recovery artifact. Do not replace them with the legacy Required-sections template.

   DO:
   - Preserve the FRESH SESSION CHECKLIST (if present above the routing block) exactly.
   - Preserve the routing block structure (begin/end markers + contents).
   - Edit routing-block rows ONLY when a plan's PLAN-STATE actually changed this session (phase_status transitioned, a plan added, or a plan archived). If nothing transitioned, don't touch the routing block.
   - Update the session-state narrative BELOW the routing block's end marker: append a new "Last checkpoint: <datetime>" summary; preserve prior-session summaries below (merge, don't overwrite).
   - **Single-source discipline (keep EVERY handoff lean for future sessions — applies to all handoff folders, every checkpoint).** The dominant cause of handoff bloat is REPETITION — the same live-state and the same verbatim directives re-recited in every session entry (left unchecked, this drives hundreds of `PAUSED` restatements and the same directives re-quoted across thousands of lines). Prevent it by keeping canonical, update-in-place zones in the durable region (durable-planning repos: between the H1 and the routing block's end marker; legacy/non-repo handoffs: the top reference sections), and writing each session entry as a short DELTA that REFERENCES those zones by id — never re-quote a directive or restate live-state that already has a canonical home.
     - **📌 Current State** — ONE copy, REWRITTEN in place each checkpoint (never appended/stacked): current phase, what is paused / awaiting / blocked / in-flight, and the immediate next action. First thing a fresh session reads.
     - **📐 Standing Directives** — verbatim user directives/goals, deduplicated, each with a stable id (`D1`, `D2`, …). Append a NEW directive once; NEVER re-quote an existing one inside a session entry — cite it as "(per D3)". This is the single canonical anti-paraphrase home; the verbatim text exists here exactly once.
     - **📜 Shipped Log** — terse, append-only, ONE line per ship: `PR# · SHA · date · headline`. No per-ship narrative in the hot file.
     - **Session delta** (what a checkpoint appends below the routing block's end marker): date + headline + what-changed + PR/SHA links + "new directives: D7" if any. It must NOT restate Current State or re-quote directives — reference them. A delta that only references canonical zones is small, and becomes cleanly demotable by the obsolescence sweep once superseded.
     - **Size budget (advisory):** aim for a hot handoff ≈ the routing block + canonical zones + the last ~5–8 session deltas. Over budget → the obsolescence sweep demotes the OLDEST superseded deltas first. Canonical zones are exempt from the budget and are never demoted.
   - **Obsolescence sweep (DEMOTE superseded history to keep the handoff lean — AUTO-ARCHIVE by default in Full mode; never delete).** The default IS to archive obsolete history every checkpoint so the hot handoff stays small. After appending this session's summary, identify superseded narrative (this session's AND older closed-loop history) so a fresh session isn't forced to read stacked dead summaries. Scope: the session-state narrative BELOW the routing block's end marker (durable-planning repos) — or, for **legacy-format handoffs** (no routing markers), the per-build/per-session history sections BELOW the hot durable zone (Current State / Standing Directives / Resume Prompt / Key Files / Directives / Open Issues / Diagnostics-runbook). The sweep MOVES eligible content to cold storage (and TAGS the uncertain) — it can never delete, and never touches the durable/hot zone (the H1, the `Last updated:` line, the Current-State block, verbatim Standing Directives, the Resume Prompt, Key Files, Diagnostics runbook), spec/plan files, the peer-review log, the backlog/plan-index, or another session's work.
     1. **Any superseded closed-loop block is a candidate — age does not gate it.** A block is a demotion candidate whether THIS session or an OLDER session appended it, as long as it clears steps 2+3 (ineligibility pre-scan + preserved-elsewhere). Refusing to touch older history would defeat the whole point — keeping the hot file lean. Other-session OWNERSHIP is still respected via step 6 (tag, don't move, on `@owner:<other>` blocks). The safety that matters is NOT age — it's steps 2 (never demote directives/secrets/single-copy) + 3 (must be superseded + preserved elsewhere) + 6 (other-owner → tag).
     2. **Mechanical ineligibility pre-scan (regex, run BEFORE any LLM judgment).** A candidate is INELIGIBLE — leave it hot, tag `@needs-review` — if grep over its text matches ANY of: `password|secret|token|api[_-]?key|sdk-key|-----BEGIN|sudo|STAYS OFF|never push|do NOT run|only on explicit|only .*(green-?light)|awaiting|PAUSED|blocked on|revert|rollback|reset --hard|force-with-lease|worktree|backup/|User goal|User directive|verbatim|Parallel session|OWNED BY|@owner:` or a literal quote mark. Also ineligible: any standing-rule construction (a modal `only|never|always|must` + an actor/system name, or "until <person> confirms/green-lights"). On ANY unrecognized structure → default INELIGIBLE.
     3. **Eligible = superseded-in-THIS-file AND closed-loop AND preserved-elsewhere.** Demote only if ALL hold: (a) a strictly-later summary in THIS SAME narrative supersedes it (same PR#/build#/task, terminal state recorded) — recency alone is not supersession, and a status update does NOT supersede a governance rule; (b) it is past-tense closed-loop (shipped/merged/DONE/deleted/clean) and survived step 2; AND (c) its durable record is POSITIVELY confirmed in a SECOND home (cite the git SHA / the backlog's Completed section / the peer-review log / a knowledge file). If a block's only copy is this handoff → KEEP. Default is KEEP.
     4. **Auto-archive by default (Full mode) — no confirmation gate.** The sweep MOVES every block that clears steps 1–3 to cold storage in the same checkpoint, automatically (the goal is a lean hot handoff every run). No propose-list, no waiting for confirmation. Two carve-outs where it still only TAGS (never moves): (a) **Compact mode** (`CHECKPOINT_MODE=compact`) — tag `@demote-candidate` only, never move, since Step 11 (commit) is skipped there and a half-written demotion must not sit uncommitted across a session boundary (per the Step 14 note); (b) **uncertain / other-owner blocks** → step 6 (tag `@needs-review`, leave hot). Everything else: move it. The mechanical non-lossy verification at Step 14 is the backstop — it greps the pre-sweep hot text for the ineligibility lexicon and FAILS the checkpoint if anything protected ended up only in cold, so auto-move is safe.
     5. **Demote = destination-first MOVE.** Append the block's FULL verbatim text (no reword/trim/summarize) to `docs/archive/handoff-cold-<repo>.md` under a dated `### <datetime> — <headline>` heading; CONFIRM the write landed (re-read or `wc -l` the archive); THEN remove it from the hot file, leaving one pointer line under `## Session history + reference`: `- <datetime> — <headline> → [archive](docs/archive/handoff-cold-<repo>.md#<anchor>)`. Never remove hot text before the archive copy is verified present (a dangling pointer = lost content). If a pointer already exists, advance its through-date — don't stack pointers. Large contiguous dead regions may be extracted in one mechanical pass (e.g. `sed -n 'A,Bp' handoff.md >> archive` then rebuild the hot file with `head`/`tail`) rather than block-by-block — same destination-first, verify-then-remove discipline.
     6. **Other-owner / shared / uncertain → tag, never move.** Blocks tagged `<!-- OWNED BY: <other> -->` / `@owner:<other>`, anything in the backlog / plan-index / peer-review log another session created, and anything you can't prove eligible → leave in place, add `@needs-review`, flag in the handoff. On a same-doc conflict, preserve BOTH sides. For the backlog specifically, completed items MOVE to the Completed section per the resolved-items bullet above — never delete another session's items.
   - **Update the `> **Last updated: <datetime>**` line directly below the `# Handoff` H1** so the file's freshness is at-a-glance visible to a fresh session before they read anything else. Use UTC ISO format `YYYY-MM-DDTHH:MMZ` (run `date -u +"%Y-%m-%dT%H:%MZ"` to get it). Include the latest commit SHA + active branch + a one-sentence headline of session state. If the handoff doesn't have this line yet, ADD it as the first non-heading content. Format example: `> **Last updated: 2026-04-29T00:59Z** (commit `abc1234` on `feat/some-branch`) — one-sentence headline of where things stand.`
   - For each plan file referenced in the routing block that had work this session: verify its `<!-- PLAN-STATE v1 -->` header reflects current state (last_commit, phase_status, next_action). Update atomically with the checkpoint commit.
   - External commitments, deliverable language, deadlines → surface in the session-state narrative AND in the relevant plan file's `next_action` or open-decisions queue.

   DO NOT:
   - Rewrite the handoff into the legacy Required-sections format.
   - Create `CURRENT.md`, `CANONICAL.md`, or any parallel top-level routing doc.
   - Paraphrase plan decisions — cite file:line to the plan file.
   - Delete the `🛑 FRESH SESSION — READ FIRST` checklist.
   - Touch the routing block if no plan's state changed.

   See [../skills/plan/SKILL.md](../skills/plan/SKILL.md) for the convention this path implements (resumable plan headers, one routing layer, cite-don't-paraphrase, verbatim goal capture, crash/token-limit resilience).

   **Legacy repos (fallback path, use ONLY if the repo hasn't adopted durable planning yet):**

   Use the Required-sections template below. Consider porting the durable-planning convention when natural — see [../skills/plan/SKILL.md](../skills/plan/SKILL.md).

   Required sections (keep concise — next session reads in 30 seconds):
     - **Timestamp** — current date/time
     - **Conversation ID** — so future sessions can reference logs
     - **Phase** — where work currently stands
     - **What's DONE** — completed items this session, with file references
     - **What's NOT DONE** — remaining items, in priority order. MUST include external commitments (emails promised, deliverables due, deadlines).
     - **Exact Next Steps** — numbered, specific, actionable steps for the next session
     - **Resume Prompt** — the EXACT copy/paste prompt the user should use to begin a new AI session, which instructs the new session to resume all relevant context and explicitly lists all relevant project files needed to understand the overall project objectives, scope, and status
     - **Key Files** — table of files the next session needs to read for full context
     - **Directives** — any standing instructions from the user that must carry forward
     - **Open Issues** — unresolved questions, blockers, or items to report to the user
     - **Persistent Context** — still-relevant context from prior sessions (carried forward until resolved). Apply the same **obsolescence sweep** as the durable-planning path (see above): only superseded, closed-loop, second-home-confirmed blocks are candidates; run the mechanical ineligibility pre-scan; demote them to `docs/archive/handoff-cold-<repo>.md` (or `<handoff-dir>/handoff-history.md` for non-repo handoffs) destination-first with a one-line pointer; everything else stays put with `@needs-review`. Never delete; never touch another session's entries.

   **All paths:**
   - Never put handoff docs in `/tmp/`, a scratch dir, or the Desktop.
   - For non-project work (health, career, etc.), put the handoff in the relevant knowledge directory.

7. **Spec Alignment (active-workstream spec files).** Spec files (e.g. under `docs/specs/`) are the architectural source of truth. Code changes shipped this session must be reflected in the owning spec file — otherwise spec and code drift and future sessions re-derive decisions from scratch.

   For every spec file the session's work touched (shipped phase, scope change, new follow-on task, discovered constraint):

   - **Header state.** Update the `<!-- status: ... -->` and `<!-- next_action: ... -->` front-matter to current reality. Valid transitions: `proposed → in_progress → phase-N-complete → all-phases-complete → archived`. Don't leave `proposed` on a spec whose Phase A shipped.
   - **Phase completion markers.** When a phase ships, add a "Phase X Status" subsection citing the merge SHA, PR number, and merge date. Example: `Phase A shipped 2026-04-23 — merged in PR #185 (5e0a347).` Cite cross-repo PRs when applicable.
   - **Spec-vs-code drift audit.** Grep for concrete numbers in the spec (jitter windows, retry caps, TTL values, cadence days) and confirm each matches the merged code. If you find a mismatch, either (a) update the spec to match merged code if the code is correct, or (b) file a follow-up task if the code drifted. Note the reconciliation inline.
   - **Invariant evidence audit.** For every invariant the spec ASSERTS (marked "must hold", "byte-identical", "always", "never"), cite committed evidence in the repo that demonstrates it holds: a smoke test, a grep-guard, a type-level guarantee, a validator, etc. If an invariant is asserted with no enforcement, either (a) downgrade the spec language from "invariant" to "goal" (aspired-to but unenforced), or (b) add enforcement this session. Unenforced invariants are a classic drift pattern — a spec says "parity" but no test verifies it, so parity silently breaks across many dimensions before anyone notices. Invariants need teeth.
   - **Internal consistency audit.** Before committing the spec, grep for any number, count, fixture name, file path, or pattern referenced in multiple places within the same spec. Confirm they agree. Common drift sources: fixture counts ("6-8" in one section vs "9" elsewhere), invariant classification totals, grep-guard patterns that don't match the actual source code being guarded (the spec's regex must literally match bytes in the source). If the spec claims a grep pattern catches duplication, actually grep for it against the real file to verify — a guard pattern that references text not present in the source would never fire.
   - **Discovered constraints.** If this session surfaced a constraint not in the spec (e.g. "library vX has no top-level `hidden` property — use `disabled: { hidden: true }` instead"), add it to the spec as a "Known constraints" bullet. Specs are the right place for non-obvious invariants that future implementers need.
   - **Sub-tasks that belong in the spec.** If a downstream task was discovered (e.g. "the admin UI must hide these 7 fields from the brand role"), add it as a spec sub-task rather than only to the backlog. The backlog is the operational queue; the spec is architectural truth — both get the entry, linked to each other.
   - **Ownership tag.** On edit, confirm the spec still has a clear owner (one workstream/session, per the Parallel-Session Coordination rules above). Never have two workstreams claim the same spec file.

8. **Plan-state validator (durable-planning repos only).** If the repo has a plan-state validator script, run it:

   ```bash
   ./scripts/validate-plan-state.sh
   ```

   Must show all plan files OK. If any fail, fix the `PLAN-STATE v1` headers (missing fields / invalid phase_status / malformed next_action) BEFORE proceeding to step 11 or 12. PLAN-STATE integrity is the foundation of cross-conversation recovery — never commit a checkpoint with broken plan headers ([../skills/plan/SKILL.md](../skills/plan/SKILL.md) §5).

9. **Peer-Review Log.** For every PR that shipped this session (merged to main or release), append an entry to the peer-review log (e.g. `docs/peer-review-log.md`) so the review workflow has a historical record and future sessions can see which review rounds each PR went through.

   Per-PR entry format (concise):
   ```
   ### PR #<num> · <short title> · <merge-SHA> · <merge-date>
   - Cross-model rounds: <N>, final verdict: <clean|mediums-only|issues>
   - PR-review-bot rounds: <N>, outcome: <clean|medium-triaged|fixed-through-round-M>
   - Cross-repo PRs: <list with SHAs if any>
   - Notable findings resolved: <1-2 sentences max>
   ```

   If zero PRs shipped this session, note it: `No PRs merged in session <date>.` Don't invent entries. (The review discipline these rounds implement lives in [Cross-Model Adversarial Review](../skills/peer-review/SKILL.md).)

10. **Cross-Repo Alignment.** If this session touched multiple repos (e.g. backend + admin UI, backend + knowledge):

    - **Confirm each repo's handoff was updated** per step 6 above. One session = one checkpoint across all repos it touched.
    - **Cross-link the SHAs.** In the spec file's Phase Status subsection and in each repo's handoff, include the merged PR SHAs from the sibling repo. Example in a backend spec: `Admin-UI visibility sub-task: PR #17 (cc1bccd) merged 2026-04-24.` Example in the admin-UI handoff: `Enables backend Phase A (backend PR #185 / 5e0a347).`
    - **Verify dependent doc state.** If repo A depends on repo B (e.g. the admin UI surfaces fields defined by the backend pipeline), quickly `git log origin/main -- <key-file>` in each repo to confirm no drift — e.g. the admin UI shouldn't reference a field the backend hasn't shipped yet.
    - **Cross-repo topology reminder:** sibling repos may push to different remotes/branches. Don't assume a single remote — confirm each repo's push target before committing.

11. **Release State & Version Control (Full mode only).**
    - **Compact mode guard:** if running in Compact mode (triggered by the pre-compaction hook), SKIP this step entirely. Leave the working tree dirty. The post-compaction session will commit when the user is ready. Rationale: pre-compaction fires mid-work; an auto-commit here would bundle half-done code changes with doc updates.
    - Run `git status` to verify changes, then explicitly stage updated docs and the handoff.
    - **Routing decision — branch+PR vs direct-to-main:** before committing, classify the doc change by the dividing-line test: *will a future session treat this doc as authoritative source-of-truth it cites or reverts?*
      - **Substantive doc artifacts** (new or materially-changed spec files, plan files, design/architecture writeups) → land via a short-lived branch + PR + squash-merge, NOT a direct commit to `main`. Flow: `git checkout -b docs/<slug>` → commit → `git push -u origin docs/<slug>` → open the PR → (let the PR-review bot confirm — see bake rule) → squash-merge with `[ci skip]` in the squash subject. Rationale: a spec/plan is cited and reverted as a unit, so it earns an atomic, linkable PR — and routing the `main` update through the merge path is cleaner than a local force-to-`main`.
      - **🔴 MANDATORY: bake every PR before opening it — the PR-review bot must only ever CONFIRM, never DISCOVER.** Automated PR-review budget is shared across all your projects; an un-baked PR that triggers multi-round re-reviews burns it fast. The draft flag does **NOT** reliably avoid this: the bot fires when you mark the PR ready (required to merge) and can post findings **asynchronously, even after merge** (observed: a draft PR the bot reviewed post-merge and flagged 3 Mediums on). So don't rely on draft to dodge review — bake so there's nothing to find. **Bake protocol (scale to PR type):** (1) **subagent adversarial review** — a skeptic agent hunts logic holes, contradictions, unhandled edges; (2) **predictive-bot pass** — run the diff against your bot's styleguide + your known-patterns file, then a "what would a pedantic bot flag" sweep; (3) **code-claim verification** — grep-verify EVERY concrete claim in the diff (cited `file:line`, hardcoded constants, "X already does Y") against the actual repo (this is where a "Service X hardcodes 5/12" claim was caught — the constant actually lived at a different call site); (4) **internal-consistency audit** — grep every number/example/fixture-count for self-contradiction. Code PRs additionally run full cross-model convergence ([Cross-Model Adversarial Review](../skills/peer-review/SKILL.md)). Doc/spec PRs: steps 1–4, non-optional. Fix everything the bake finds, THEN open the PR. **Re-bake the FINAL diff, not an earlier state:** the fixes you apply in response to the first bake can themselves introduce new issues that never got reviewed — at minimum re-run steps 3–4 on the post-fix diff. (Observed: a math bug was *introduced while fixing other bake findings* and shipped to the PR un-baked; the bot caught it on a second round.)
      - **Routine bookkeeping** (handoff syncs, backlog status ticks, peer-review-log entries, release-tracking files) → commit **directly to `main`** with `[ci skip]`. These are snapshots of current state — nobody reviews them and you'd never revert "the checkpoint," so PR ceremony is pure overhead on the highest-frequency path.
      - When a single checkpoint touches BOTH (e.g. a new spec plus a handoff sync): land the substantive artifact via its own baked branch+PR first, then commit the bookkeeping direct to `main`. Keep them in separate commits regardless.
      - **Obsolescence sweep commits (if any block was demoted per Step 6):** the handoff edit AND the `docs/archive/handoff-cold-<repo>.md` append are routine bookkeeping — commit them TOGETHER in the same `chore: Checkpoint save [ci skip]` commit so a pointer is never committed without its archive target. **DISABLE the sweep's move step entirely in Compact mode** (Step 11 is skipped there; a half-written demotion must never sit uncommitted across a session boundary where a parallel session could stomp the dirty pair and sever a pointer) — in Compact mode, tag `@demote-candidate` only, never move.
      - Compact mode is unaffected: it skips this step entirely and leaves the tree dirty (see Compact-mode guard above).
    - **Commit-message convention:**
      - **Checkpoint commits** use `chore: Checkpoint save [ci skip]` (the generic prefix). Keeps checkpoint commits distinct from plan-workstream commits in `git log --grep`.
      - **Plan-workstream commits** (when phase work shipped this session AND is being committed here, not in a separate atomic commit) use `<plan-slug>(phase-N): <summary>` per the plan-state commit convention. Example: `product-category-enforcement(phase-B): canonical inference module + kill local copies`. Allows `git log --grep='<plan-slug>'` reconstruction of workstream execution.
      - When in doubt: keep phase work in its own commit (plan-prefix) and use `chore: Checkpoint save [ci skip]` for the pure checkpoint sync.
    - Execute the commit. A branch-lock hook (if installed) may allow `chore:` for documentation while blocking it if source code is staged on `main`.
    - If the commit fails due to a branch-lock violation (source code on `main`), HALT. Ask the user to checkout a feature branch before retrying.
    - Verify the backlog and QA-case list reflect the exact state of the codebase.
    - For durable-planning repos: confirm the plan-state validator still passes after commit.
    - If the session resulted in a deployed/tested release, verify that the release version numbers are properly bumped.
12. **Sync Agent Workflows.** If you maintain a version-controlled agent-config directory (skill files, workflow `.md` edits, ledgers), sync it now so this session's workflow changes are committed. Config dirs that live on a synced drive (iCloud/Dropbox/etc.) need an explicit, atomic write — don't leave them half-written.

13. **Confirm to the user.** List the handoff file paths and the resume command for each. For durable-planning repos: the resume command is typically "Read the handoff from the top. Follow the FRESH SESSION CHECKLIST exactly. Then execute the next action it names."

14. **Final Integrity Pass (Mandatory).** Before declaring the checkpoint complete, run these concrete verifications. Treat each as a gate — if any fails, fix it before step 15. A checkpoint that hands a confused next-session the bag isn't a checkpoint, it's a liability.

    **14a. Handoff completeness (read top-to-bottom, not skim).**
    - `Last updated:` header line present, in UTC ISO format, with current commit SHA and active branch.
    - Every external commitment from this session captured (emails sent, deliverables promised, deadlines, recipient names) — verbatim, not paraphrased.
    - Every user directive issued this session quoted verbatim with date stamp (`**User directive (YYYY-MM-DD):** "<exact quote>"`).
    - Resume command at end is copy-pasteable and names every file the next session needs to read for full context (handoff, relevant specs, backlog, plans, release-tracking).
    - No vague references — every PR#, SHA, build#, spec ID mentioned in the narrative actually resolves (spot-check 2-3 via `git log` / `gh pr view`).
    - **Obsolescence sweep is non-lossy (mechanical check, NOT LLM self-certification).** If any block was demoted this session: (1) each demoted block resolves at its `docs/archive/handoff-cold-<repo>.md` anchor (spot-check 2-3); (2) the hot file carries a live pointer to each; (3) grep the PRE-SWEEP hot text for the Step-6 ineligibility lexicon (`password|secret|token|api-key|STAYS OFF|never push|do NOT run|only on explicit|awaiting|PAUSED|revert|rollback|worktree|User goal|User directive|verbatim|Parallel session|@owner:`) and assert EVERY hit still resolves in the hot durable zone OR a named second home — if any hit now lives ONLY in cold, the checkpoint FAILS: restore that block to hot and re-tag `@needs-review`. It is a grep, not a judgment — the model that chose to demote does not self-certify it.
    - **No staleness left hot.** Nothing in the session-state narrative contradicts the routing block or the `Last updated:` SHA. Superseded same-session summaries are demoted or `@demote-candidate`-tagged, not silently stacked.

    **14b. Spec accuracy (touched specs only).**
    - For each spec file modified or referenced this session: `<!-- status: ... -->` field matches reality. No `proposed` on a spec whose Phase A shipped. No `in_progress` on an all-phases-complete spec.
    - Every shipped phase has a "Phase X Status" subsection citing merge SHA, PR#, merge date.
    - Cross-repo PR SHAs cited where applicable.
    - Spot-check 1-2 concrete numbers per spec (retry caps, TTL values, fixture counts) against merged code via `grep`.

    **14c. Status accuracy across operational docs.**
    - Backlog: completed items moved to the Completed section. No item still listed in "Active" that this session shipped.
    - Release-tracking doc (if touched): shipped builds moved to the Completed table. In-flight rows reflect the actual deployed inventory, not a config file's intent.
    - Peer-review log: every PR shipped this session has an entry (cross-check against the session's actual PR list from `gh pr list --state merged --search 'merged:>=YYYY-MM-DD'`).
    - QA-case list (if applicable): new test scenarios from shipped features documented ([../templates/QA_CASES.md](../templates/QA_CASES.md)).

    **14d. Git hygiene (no orphans, no lingering dirt).**
    - `git status` — working tree clean, OR explicitly note the dirty files in the handoff with rationale (e.g. "WIP on spec draft, intentionally uncommitted").
    - `git branch --no-merged main` — any local branches with commits not in main? For each: confirm either (a) it's in a still-open PR (cite PR#), (b) it's deliberately archived/parked (note in handoff), or (c) it should be deleted now.
    - `git branch --merged main | grep -v '^\* main$'` — any local branches whose work is fully in main? Delete them or note why kept. (Use `git branch -d <name>`, not `-D`, so git refuses if state diverges.)
    - Current branch matches expectation for next session (typically `main` for personal projects, or the active feature branch for in-flight work).
    - For multi-repo sessions: repeat 14d in EACH touched repo.
    - `git log origin/main..HEAD` — any local commits not pushed? Push them or note why holding.

    **14e. Symlink + workflow infrastructure sanity.**
    - If any command/skill/config symlinks were modified this session, run `ls -la` to confirm they resolve. Broken symlinks silently break slash-command discovery in future sessions.

    Output the result of 14a–14e as a brief "Final integrity pass:" block in the user-facing confirmation (one line per check: ✓ or what was fixed). If anything was fixed in this pass, commit those fixes via the same `chore: Checkpoint save [ci skip]` convention.

15. **Compact the session context.** As the very last action — after step 14's integrity block is printed and any fix-commits are pushed — instruct the user to compact the conversation context for the next session. If your harness exposes a built-in compaction command that can't be invoked programmatically from a workflow, the assistant's final user-facing line MUST be exactly that instruction and nothing else, so the next step is zero-friction: the user runs it and the next session starts with a clean compacted context plus the integrity-verified handoff to read.
