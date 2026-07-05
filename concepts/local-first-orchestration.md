---
description: Why/overview for local-first orchestration — how to push the bulk of delegable work to cheap/local models while a frontier model orchestrates and verifies, without giving up quality. Covers the broker, the router, the supervision protocol, and the economy ladder.
---

# Local-First Orchestration

How to cut the cost of an AI-first practice without cutting its quality: let **cheap or local models do the volume**, and keep a **frontier model as the orchestrator and verifier**. Governing idea: **quality is set by who *decides*, cost is set by who *drafts*** — so move the drafting down the tier ladder and keep the deciding at the top.

This is the **overview** — the *why*, and the four pieces that make aggressive delegation safe. It pairs with [Engineering Principles](../workflows/engineering-principles.md) (the model-tiering rule) and the subagent execution loop in [The AI Development Lifecycle](ai-dev-lifecycle.md) (fresh subagent per task, cheap model for mechanical work). Where this borrows from named tools, they're credited in [CREDITS.md](../CREDITS.md).

## The two axes people conflate

"Spend less on AI" is usually read as one dial — use a weaker model. That collapses two independent choices:

1. **Orchestrator tier** — how capable is the model that plans, decomposes, reviews, and decides? Turning this down degrades everything.
2. **Delegation depth** — how much of the *labor* (drafts, summaries, classification, extraction, first-pass review, bulk transforms, bounded code-gen) leaves the orchestrator for a cheaper worker? Turning this **up** cuts cost with no quality loss — *if* the orchestrator still authors the spec and verifies the result.

Local-first orchestration moves the second dial hard while holding the first at the top. The frontier model's token **volume** drops; its **quality** never does. Everything below is the machinery that makes that trade safe.

## The safety problem, and why it needs a mechanism

Naive delegation fails two ways, and each has a dedicated layer below:

- **Resource collisions.** Several sessions (or jobs, or teammates) demanding large local models at once will thrash memory — on a single machine that means swapping, stalls, or a kernel panic. → **Layer 1, the broker.**
- **Silent quality rot.** A worker returns something plausible-but-wrong and it flows downstream unchecked. → **Layer 3, the supervision protocol.** Deciding *which* work is even eligible is → **Layer 2, the router.**

The four pieces: a **broker** that owns all local-model lifecycle, a **router** that decides what may leave the frontier model, a **supervision protocol** that verifies what comes back, and an **economy ladder** that tunes how aggressive the whole thing is.

## Layer 1 — the broker (one door for all local inference)

Run **one** local-inference gateway that owns *all* local-model lifecycle and request queueing. Every consumer — interactive sessions, scheduled jobs, batch pipelines, other people's apps — calls the same OpenAI-compatible endpoint. Nothing else starts a model directly.

This is the collision fix **by construction**: a single admission queue serializes generations through one memory accounting, so two callers can never race two big models onto the hardware at once. Give the broker:

- **A memory ledger + admission control.** Before loading a model it checks the budget against what's already resident; it **fails closed** (refuses, cleanly) rather than over-committing. Co-resident small models are fine when they fit; a model that can't fit gets a queue slot or a clean "busy," never an OOM.
- **QoS priority classes.** Rank callers — e.g. *interactive-human* > *scheduled-jobs* > *shared-user* > *background*. A higher class jumps the **queue order**; it never kills a running generation. This is what lets one machine be shared without one workload starving another.
- **Per-caller keys.** Each caller authenticates with its own key, which maps to a QoS class and to telemetry attribution.
- **Model resolution by role, not by name.** Callers ask for `role:reasoning` / `role:code` / `role:classification`, not a specific checkpoint. The role points at *whatever the current best model is* (see [Model currency](#model-currency)), so upgrades propagate to every client instantly and no caller hard-codes a stale model.

**Build vs. buy.** Off-the-shelf proxies (llama-swap for hot-swap/TTL, LiteLLM/Portkey for gateway features) cover much of this; adopt one if it fits. Build a thin own only when you need something they don't — mixed runtimes, preemption of a foreign local-model app, batch keep-sets, custom QoS. Either way the invariant holds: **exactly one process owns the hardware.**

## Layer 2 — the router (what is even eligible to leave)

A **deterministic table**, not a model's judgment call, decides where each subtask goes. Key it on **task-type × context-size × risk-domain** → one of three verdicts:

| Verdict | Meaning |
|---|---|
| **local-first** | Route to a local role; the orchestrator verifies the result. Drafts, summaries, classification, extraction, research digestion, first-pass code review, bulk transforms, and **bounded** code-gen (a single function/module against an explicit spec + tests, scaffolding, mechanical refactors/codemods, doc comments). |
| **frontier-only** | The orchestrator does it itself. Multi-file design, API shape, concurrency, security-sensitive code, final verification, client-facing final output. |
| **never-local** | **Hard refuse** to route anywhere but the frontier model — no mode overrides it. Financial, legal, and safety judgments; anything that mutates a calendar, sends mail, or takes an irreversible outward action. |

Design rules that keep it trustworthy:

- **Start static.** A hand-written table is auditable and testable — you can unit-test that `never-local` types are refused in every mode. Add *learned* routing (a classifier that predicts whether the cheap model will suffice) only if the static table plateaus; it's an optimization, not a foundation.
- **The `never-local` list is a hard floor,** enforced in code, independent of the economy mode. It is the first thing a reviewer should check.
- **Tune from telemetry, not vibes.** The weekly review (below) proposes route changes from measured outcomes.

## Layer 3 — the supervision protocol (the quality guarantee)

This is the layer that *permits* aggression everywhere else. A delegated subtask is a contract:

1. **The orchestrator supplies** the task-type, the payload, and **explicit acceptance criteria** — the same rigor as authoring a spec + tests for a subagent.
2. **The router picks** the model (Layer 2).
3. **The worker returns** its output plus a self-reported confidence and any verification artifacts.
4. **The orchestrator verifies** the output against the acceptance criteria. This verification stays at frontier quality — **it *is* the quality floor.**
5. **On failure, re-route or escalate — bounded.** At most **two** retries, then hand the task up to the frontier model. Escalation is the orchestrator's job; the delegation tool never silently falls back to a more expensive model behind your back.

Two corollaries worth stating outright:

- **Bounded code-gen is safe *because* of this loop.** Letting a cheap model write a function is fine when the frontier model wrote the spec and the tests and reviews the diff — and the normal release gate still stands between it and production. Without the loop, don't.
- **Verification quality = orchestrator quality.** If you swap a weaker model into the orchestrator seat (see the fully-local option below), you have lowered the quality floor for *everything*. Never ship code whose only reviewer was a local model — keep a frontier verifier on anything that merges.

## The economy ladder (how aggressive, as one setting)

Expose the aggressiveness as a small ladder the whole system reads — a **routing bias**, not prose each session re-interprets. A workable shape:

| Mode | Posture |
|---|---|
| **0 — frontier-first** | Local only for always-local background jobs. Highest cost, simplest. |
| **1 — balanced** | Cheap-tier discipline + opportunistic local drafts/review. |
| **2 — local-bulk** *(sensible default)* | Every delegable subtask routes local-first; the frontier model stays full-quality as orchestrator/verifier. Sub-dials tune it: *lite* (local only for clearly-safe types) · *std* (the full local-first list) · *max* (local-first for everything not `never-local`, tolerate a retry before escalating). |
| **3 — austerity** | Mode 2 + a cheaper orchestrator tier + defer nice-to-haves. |

The mode parameterizes the router (local-first aggressiveness, escalation patience); it does not change the `never-local` floor and does not lower orchestrator quality except at the explicit top of the ladder. **Orchestrator-agnostic by construction:** the same stack works under any capable orchestrator — a different frontier tier, or even a local model for low-stakes work — because the mode governs routing regardless of who sits in the seat. The one caveat is the verification rule above.

## Model currency

Roles only stay cheap-*and*-good if the model behind each role keeps up. Automate it: on a cadence, discover newer candidate models, **size-gate** them (fail closed on too-big/unknown), evaluate on a fixed set of **golden tasks**, and **promote only a clear win** — with an absolute floor on critical tasks and the prior model retained for one-command rollback. Because callers address **roles**, a promotion propagates everywhere (including shared users) with no client change. Report reclaimable disk on the same cadence; never auto-delete a model that's an incumbent, a rollback, or in use.

## Measurement — the two numbers to watch

Instrument every delegated call with its task-type, route, and the verification verdict. Two numbers tell you if the system is working:

- **% of subtasks served locally** — should rise as you tune the router.
- **Escalation rate** — should stay flat and low. A rising escalation rate means the router is sending work the cheap models can't do; retune, don't just absorb the retries.

Success is the pair moving the right way together: local share **up**, escalation **flat**, frontier spend **down**. If local share rises but escalations spike, you've moved cost from tokens to wasted round-trips — a worse place.

---

**The map, one line each:** the **broker** makes sharing hardware safe · the **router** decides what may leave the frontier model · the **supervision protocol** verifies what comes back, bounded to two retries · the **economy ladder** tunes how hard you push · **currency** keeps the local roles good · and **two telemetry numbers** tell you it's working. Pair with [the lifecycle](ai-dev-lifecycle.md) and the model-tiering rule in [Engineering Principles](../workflows/engineering-principles.md).
