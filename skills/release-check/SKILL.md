---
name: release-check
description: Pre-release checklist that verifies build, lint, type, tests, secrets, and git hygiene before a branch is ready for merge. Run this before creating a PR.
allowed-tools:
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git status:*)
  - Bash(git show:*)
  - Bash(cat:*)
  - Bash(grep:*)
  - Bash(find:*)
  - Bash(build:*)
  - Bash(lint:*)
  - Bash(typecheck:*)
  - Bash(test:*)
  - Read
---

# Pre-Release Check Protocol

Run this before creating a PR or deploying to staging. This is a go/no-go gate.

It is a fast local sanity check, NOT a replacement for the full review pipeline. Run it before `/peer-review` so you don't waste an adversarial-review round (or a PR-review-bot round) on a trivially broken branch. For the formal path, see `/peer-review` → `/release-prep` → `/release-prod`.

## Step 1: Detect Repo Type
Read your project's agent-instructions file (e.g. `CLAUDE.md`) and the manifest (`package.json`, `pyproject.toml`, `pubspec.yaml`, `Cargo.toml`, etc.) to determine:
- What build/lint/test commands are available
- What branching model applies (main, release, etc.)
- What deploy targets exist

## Step 2: Code Quality Gates

### Build
- [ ] Run the build command
- [ ] Zero build errors

### Lint
- [ ] Run the lint command
- [ ] Zero lint errors (warnings are acceptable if pre-existing)

### Type Checking
- [ ] Run the type checker if applicable
- [ ] Zero type errors

### Tests (if available)
- [ ] Run the test suite
- [ ] All tests pass
- [ ] Note test count and any skipped tests

## Step 3: Documentation Sync

Check that these files reflect the current state of the branch:

- [ ] Handoff/state file — updated with current branch status
- [ ] Backlog — any completed items marked done, any new items added
- [ ] Agent-instructions file (`CLAUDE.md`) and your bot's sacred-rules file — updated if architecture or build commands changed
- [ ] QA-cases file — updated if new edge cases discovered
- [ ] Spec files — any relevant spec updated with implementation notes

## Step 4: Git Hygiene

- [ ] `git status` shows clean working tree (no uncommitted changes)
- [ ] `git log --oneline main..HEAD` shows coherent commit history
- [ ] Branch name follows convention (`feature/`, `fix/`, `chore/`)
- [ ] No merge conflicts with target branch (`git merge-base --is-ancestor`)

## Step 5: Deploy Safety

- [ ] No `.env` files or secrets in the diff (grep the diff for tokens, keys, passwords)
- [ ] No private/internal docs in deploy artifacts — keep your agent-instructions file, your bot's sacred-rules file, handoff/state files, and `docs/` out of shipped builds
- [ ] Build mode matches target environment (staging build for staging, etc.)

## Output Report

```markdown
# Release Check: [branch-name] → [target-branch]

## Gate Status: [PASS ✅ / FAIL ❌ / WARN ⚠️]

| Check | Status | Notes |
|-------|--------|-------|
| Build | ✅/❌ | |
| Lint | ✅/❌ | |
| Types | ✅/❌ | |
| Tests | ✅/❌ | X passed, Y skipped |
| Docs Sync | ✅/❌ | |
| Git Clean | ✅/❌ | |
| Deploy Safe | ✅/❌ | |

## Blockers (if any)
[list of items that must be fixed before merge]

## Recommendation
[PROCEED / FIX AND RE-CHECK]
```
