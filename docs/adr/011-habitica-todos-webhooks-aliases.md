---
type: adr
project: iris
number: 011
status: accepted
updated: 2026-11-16
---

# 011. Habitica: todos only, webhooks inbound, alias-based identity

## Context

Habitica has habits, dailies, todos and rewards; only todos map to one-shot tasks. It supports user webhooks (`taskActivity`), which are not signed.

## Decision

- Iris syncs only `type: "todo"`.
- **Outbound:** `POST /api/v3/tasks/user` with `alias: "iris-<taskId>"`, `PUT /api/v3/tasks/{id}` for fields, `POST /api/v3/tasks/{id}/score/up` on completion, `score/down` on reopen, and `DELETE` on deletion. Headers are `x-api-user`, `x-api-key` and `x-client: <userId>-iris`. The client throttles to ≤ 25 req/min.
- **Inbound:** a `taskActivity` webhook (created/updated/deleted/scored) to `POST /v1/ingest/habitica/{HABITICA_WEBHOOK_SECRET}`.
- **Priority mapping:** `NONE↔0.1`, `LOW↔1`, `MEDIUM↔1.5`, `HIGH↔2`.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Completing a task anywhere grants Habitica rewards, and the alias makes webhook echoes resolvable before the link commits.
