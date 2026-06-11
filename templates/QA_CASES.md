# QA Cases

> Run **all relevant cases** before any change is "done." When a bug is found, fix it, then re-run all relevant cases — not just the one that failed. **This list only grows:** every new failure mode discovered becomes a permanent case here (Principles §6).

## How to use

- Each case tests a **business outcome or a failure mode**, not "does it compile."
- Cover the unhappy paths first: boundaries, empty/null, error states, permission denials, state transitions, integration seams.
- "Last run" + "Status" are updated each QA pass so it's obvious what's stale.

## Cases

| # | Area | Scenario / steps | Expected result | Last run | Status |
|---|---|---|---|---|---|
| 1 | `<area>` | `<steps to reproduce the scenario>` | `<what must happen>` | `<date>` | ☐ pass / ☐ fail |
| 2 | `<area>` | `<...>` | `<...>` | | |

## Added-from-failures log

> When QA (or production) reveals a new failure mode, add the case above and note it here with the date — so the list's growth is auditable.
>
> *This log is the **bug-triggered half** of a self-improvement loop; the session-triggered half is checkpointing (see the lifecycle). Same goal: the system gets measurably better each cycle.*

- `YYYY-MM-DD` — `<new case # and what failure it guards against>`
