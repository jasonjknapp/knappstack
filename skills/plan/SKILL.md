---
name: plan
description: Durable planning for multi-session, multi-agent work. Writes a plan file with a PLAN-STATE v1 header, enforces one routing layer, cite-don't-paraphrase, and verbatim goal capture so decisions survive across sessions without drifting. Invoked as `/plan`.
---

# /plan — Durable Planning (State That Survives Across Sessions)

Multi-session, multi-agent work drifts: decisions get paraphrased a little differently each time until they diverge from what was decided; parallel "status" docs multiply until no one knows which is canonical; a session ends and the next can't tell what's done. This is the convention that prevents all of it. It's the backbone under the [Staged, Gated Release](../../workflows/release-prep.md) path, and what makes Principles §12 (agents crash — plan for it) and §14 (rules decay — anchor them to intent) real.

This skill is the operational home of the Durable Planning convention. Install per [SETUP.md](../../SETUP.md).

## 1. Every plan file carries a state header

Open each plan file (`docs/plans/<slug>.md`) with a machine-checkable header — the single source of truth for where that workstream stands:

```
<!-- PLAN-STATE v1 -->
current_phase: <phase-id>
phase_status: proposed | in_progress | awaiting_review | complete
last_commit:  <sha>
next_action:  <one line — exactly what the next session should do first>
-->
```

> [!IMPORTANT]
> **Update the header atomically with the work.** The commit that ships a phase is the same commit that flips `phase_status`, updates `last_commit`, and rewrites `next_action`. State updated separately from the work it describes is already lying.

A ready-to-copy plan skeleton lives in [`../../templates/PLAN.md`](../../templates/PLAN.md).

## 2. One routing layer — never parallel ones

There is exactly **one** place that says "here's the current state and the active workstreams": the handoff/state doc + the live index of plans + the plan headers. **Never create parallel routing docs** — no `CURRENT.md`, `STATUS.md`, `CANONICAL.md`. Every parallel doc is a future contradiction. The handoff narrative is for *session* state; *plan* state lives in the headers — don't restate plan state in the handoff body or it drifts.

## 3. Cite, don't paraphrase

When you reference a past decision, **link the plan file and quote the line.** A paraphrased summary diverges across sessions; a verbatim citation does not. This single habit kills most decision drift.

## 4. Capture goals verbatim

When a stakeholder states a goal or constraint, copy it **verbatim** into the plan's Why section, dated and attributed:

> **User goal (YYYY-MM-DD):** "*exact quote*"

That quote is the anti-paraphrase anchor everything else is checked against.

## 5. Validator before push

A small linter over every plan header (valid `phase_status`, present `next_action`, parseable `last_commit`) catches the headers people forget to update. Run it before pushing if you touched any plan file. Cheap insurance.

## 6. The recovery test

Durable state lives in version control + these files, never only in the agent's context window. The test: **if this session died right now, could a fresh session on a different machine pick up from `next_action` and be correct?** If not, the state isn't durable yet — commit and write it down before you need it (Principles §12). Pairs with the lifecycle's [Checkpointing](../../concepts/ai-dev-lifecycle.md#checkpointing) step, which closes each session by updating the state doc + plan headers.
