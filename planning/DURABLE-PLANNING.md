# Durable Planning — State That Survives Across Sessions

Multi-session, multi-agent work drifts. Decisions get paraphrased a little differently each time until they quietly diverge from what was actually decided; parallel "status" docs multiply until no one knows which is canonical; a session ends and the next one can't tell what's done. This is the convention that prevents all of it.

It's the backbone under the [AI Development Lifecycle](../lifecycle/AI-DEV-LIFECYCLE.md), and it's what makes Principles §12 (build for recovery) and §14 (rules age) real.

---

## 1. Plan files carry a state header

Each plan file opens with a small, machine-checkable header — the single source of truth for where that workstream stands:

```
<!-- PLAN-STATE v1 -->
current_phase: <phase-id>
phase_status: proposed | in_progress | awaiting_review | complete
last_commit:  <sha>
next_action:  <one line — exactly what the next session should do first>
-->
```

**Update the header atomically with the work.** The commit that ships a phase is the same commit that flips `phase_status` and rewrites `next_action`. State that's updated separately from the work it describes is already lying.

## 2. One routing layer — never parallel ones

There is exactly **one** place that says "here's the current state and the active workstreams": a state/handoff doc plus a live index of plans. 

- **Never create parallel routing docs** — no `CURRENT.md`, `STATUS.md`, `CANONICAL.md` sprawl. The handoff + the plan index + the plan headers *are* the routing layer. Every parallel doc is a future contradiction.
- The handoff narrative is for *session* state. *Plan* state lives in the plan headers. Don't restate plan state in the handoff body — it will drift.

## 3. Cite, don't paraphrase

When you reference a past decision, **link the plan file and quote the line.** A paraphrased summary diverges across sessions; a verbatim citation does not. This single habit kills most decision drift.

## 4. Capture goals verbatim

When a stakeholder states a goal or constraint, copy it **verbatim** into the plan's "why" section, dated and attributed:

> **User goal (YYYY-MM-DD):** "*exact quote*"

That quote becomes the anti-paraphrase anchor everything else is checked against.

## 5. A validator (recommended)

A small script that lints every plan header (valid `phase_status`, present `next_action`, parseable `last_commit`) and runs before push catches the headers people forget to update. Cheap insurance.

## 6. Crash / token-limit resilience

Durable state lives in **version control + these files**, never only in an agent's context window. The test: if this session died right now, could a fresh session on a different machine pick up from `next_action` and be correct? If not, the state isn't durable yet. Commit and write it down before you need it.

---

## Why this beats "just keep good notes"

Notes are prose; prose drifts. This convention makes state **structured** (headers), **singular** (one routing layer), **verbatim** (cite don't paraphrase), and **co-located with the work** (atomic updates). Those four properties are what let many sessions — human or agent — touch the same workstream over weeks without the plan rotting.

See [`../templates/PLAN.md`](../templates/PLAN.md) for a ready-to-copy plan skeleton.
