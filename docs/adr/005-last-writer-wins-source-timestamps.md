---
type: adr
project: iris
number: 005
status: accepted
updated: 2026-11-16
---

# 005. Conflict resolution: record-level last-writer-wins on source timestamps

## Context

The same task can be edited on two platforms between syncs.

## Decision

Every inbound change carries `sourceUpdatedAt`, the platform's own modification time (Taskwarrior `modified`, Google `updated`, Habitica `updatedAt`). Iris stores the winning edit's time as `Task.lastEditAt`. An inbound change with `sourceUpdatedAt < lastEditAt − 2s` (skew tolerance) is `REJECTED_STALE`. Iris does not apply it and enqueues a push back to the origin, when the origin is a push target, so that platform converges to canonical.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

The rule is simple and deterministic. Edits to different fields on two platforms within one sync interval lose the older edit; field-level merge is a possible future ADR. Clock skew beyond 2 s between PCs can misorder edits.
