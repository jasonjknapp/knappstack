# Credits & influences

This playbook is **predominantly original work** — developed with AI assistance and hardened across real product and consulting delivery. Where a practice came from a nameable source, it's named. **If you see something of yours uncredited, that's an oversight, not a claim — open an issue and I'll fix it.**

## What genuinely shaped this

- **Anthropic — Claude Code & context-management research.** The context discipline (clear/compact, context priming, tiered load-on-demand context, and the "context rot / catastrophic forgetting" failure modes the rules guard against) builds on Anthropic's Claude Code and its published guidance.
- **[obra/superpowers](https://github.com/obra/superpowers)** (Jesse Vincent — MIT). Four development-loop disciplines are adapted, with thanks: the **systematic-debugging** method (root-cause-before-fix, the four phases, "3 failed fixes → question the architecture"); **subagent-driven execution** (fresh subagent per task, spec-review then quality-review, status protocol, model-tiering); the **brainstorming hard-gate** (no code before an approved design); and **bite-sized / no-placeholder task authoring**. Re-expressed in this playbook's voice; the structure and ideas are Jesse's.
- **Kris Skrinak — [`skrinak/ContextEng`](https://github.com/skrinak/ContextEng), [`skrinak/Simple-AI-DLC`](https://github.com/skrinak/Simple-AI-DLC).** Four agent-specific principles credit *concepts* from Kris's work — treating permissions as architecture, building for agent crashes, verifying the harness (not just the work), and anchoring rules to intent over specifics. Wording original; ideas his. (His repos carry no license, so only ideas are used — in our own words.)

## Influences & further reading

- **[humanlayer/12-factor-agents](https://github.com/humanlayer/12-factor-agents)** — principles for production-grade LLM software; aligned with this playbook's principles spine (own your context window, own your prompts, small focused agents).
- **[github/spec-kit](https://github.com/github/spec-kit)** — GitHub's spec-driven-development toolkit; the same intent → spec → plan → implement arc.
- The **context-engineering** body of work — writing / selecting / compressing / isolating context.
- **"12-factor agents"** and **spec-driven development** as a broader school — boring primitives over clever orchestration.

## Honest note on lineage

Generalized from a larger private workflow set that also draws on frameworks **not used in this playbook** — Sahil Lavingia's minimalist lens, Barbara Minto's SCQA and the U.S. Army's BLUF for executive writing, `blader/humanizer` for communication. Named for completeness; none are embedded in the practices here.
