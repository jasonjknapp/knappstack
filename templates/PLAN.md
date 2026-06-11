<!-- PLAN-STATE v1 -->
<!-- current_phase: <phase-id>
     phase_status: proposed | in_progress | awaiting_review | complete
     last_commit:  <sha>
     next_action:  <one line — exactly what the next session does first>
-->

# Plan: <title>

## Why

**User goal (YYYY-MM-DD):** "*verbatim quote of the goal/constraint as the stakeholder stated it*"

One paragraph: the outcome this serves and how we'll know it worked.

## Phases

| Phase | Goal | Status | Exit criterion |
|---|---|---|---|
| A — `<name>` | `<what>` | proposed | `<how we know it's done>` |
| B — `<name>` | `<what>` | proposed | `<...>` |

## Open questions / decisions needed

- [ ] `<question>` — owner: `<who>`

## Resume protocol

If a session is cut off mid-phase: read this header's `next_action`, then `<the durable artifacts to reconstruct done-vs-not — e.g. git log of the feature branch, the task list>`. Resume at the first incomplete item; re-verify the last commit built/ran before continuing.
