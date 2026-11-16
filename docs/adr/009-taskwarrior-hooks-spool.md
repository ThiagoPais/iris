---
type: adr
project: iris
number: 009
status: accepted
updated: 2026-11-16
---

# 009. Taskwarrior hooks: `on-add` + `on-modify`, spool on failure

## Context

Iris must react both when a task is **created** and when it is **modified** (including `done` and `delete`). Hooks run synchronously inside the `task` CLI.

## Decision

- One Python 3 (stdlib-only) script, `iris_hook.py`, symlinked as `~/.task/hooks/on-add.iris` and `~/.task/hooks/on-modify.iris`. The mode comes from `basename(argv[0])`.
- The hook always echoes the required JSON on stdout and exits `0`, and it never blocks the task.
- **Spool-first:** the hook never touches the network. It appends the event to `~/.local/state/iris/spool.jsonl` (under an `fcntl` lock) and starts a **detached** `iris_agent.py flush` (new session, all stdio on `/dev/null`), then returns. The flush POSTs spooled events in order and removes only the ones Iris acknowledged. The timer-driven agent flushes again before every pull, so events survive Iris being offline.
- Tasks tagged `+nosync` and recurrence templates (`status: recurring`) are not sent.
- The agent applies remote changes with `task rc.hooks=off rc.confirmation=off import`, so applying them can never re-trigger the hooks.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Hook cost is Python startup plus one file append (≤ 100 ms) whether Iris is up or down. Events are never lost, and Iris writes to Taskwarrior do not loop back. Latency to Iris is the time for the detached flush to run (well under 1 s when Iris is up).
