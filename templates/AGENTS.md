# <PROJECT NAME> — Agent Guide

> **`AGENTS.md` is the cross-tool standard instruction file** (read natively by
> Claude Code, Codex, and other agent CLIs; Gemini reads it via a symlink). Make
> THIS file the single canonical source, and make each tool's own filename a
> **symlink to it** so there is exactly one copy to maintain:
>
> ```sh
> ln -s AGENTS.md CLAUDE.md      # Claude Code
> ln -s AGENTS.md GEMINI.md      # Gemini CLI
> ```
>
> Never fork these into separate hand-edited copies — that is how instruction
> drift starts. A drift check (see the repo README, "One canonical instruction
> file") can verify every surface still resolves to this file.

> Project constitution. Loaded at the start of every agent session. Keep it tight — this is "always needed" context; push "sometimes needed" detail to `docs/` and load it on demand.
>
> Replace every `<placeholder>`. Delete guidance lines once filled in.

## Read first, in this order

1. **This file** — hard rules, facts, conventions.
2. **`<handoff/state doc>`** — current state + the live index of active plans.
3. **`<backlog doc>`** — deferred work.
4. For an active workstream: its plan file under `<plans dir>` — the state header tells you `current_phase` / `phase_status` / `next_action`.

## Hard rules

> The non-negotiables. Keep them few and write them around *intent*, not brittle literals (see Principles §14).

- **Brand / naming:** `<exact capitalization and spelling — the thing to never get wrong>`.
- **Production safety:** never deploy or mutate production without explicit approval. Present the plan, dry-run first.
- **Secrets:** never commit credentials/tokens/keys. They live at `<secrets location, gitignored>`.
- **Scope / IP:** `<what this repo must never contain — e.g. client code, personal data>`.
- `<any project-specific "never do this">`

## Always-current facts

| Fact | Value |
|---|---|
| Cloud account / project | `<id>` |
| Hosting / deploy target | `<value>` |
| Primary datastore | `<value>` |
| Error tracker | `<value>` |
| Key URLs (prod / staging) | `<value>` |
| `<other load-bearing constant>` | `<value>` |

## Stakeholders

| Person | Role | Notes |
|---|---|---|
| `<name>` | `<role>` | `<how to work with them>` |

## Folder map

| Folder | Holds |
|---|---|
| `<dir>` | `<what>` |

## Workflow conventions

- **Commit often**, with descriptive messages. A pushed commit is the only proof a step is done.
- **Plan state** lives in plan-file headers; update them atomically with the work (see Durable Planning).
- **Code standards:** `<language/version, typing, lint/format commands>`.
- **Before marking work done:** run lint + typecheck + the relevant tests; run all relevant QA cases.
- **End every working session** by updating the state doc + any plan headers you touched.
- **Cite, don't paraphrase** past decisions — link the plan file and quote the line.
