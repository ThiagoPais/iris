---
type: adr
project: iris
number: 003
status: accepted
updated: 2026-11-16
---

# 003. Per-platform link table instead of nullable ID columns

**Supersedes:** `TaskMapping.taskwarriorUuid/googleTaskId/habiticaId/googleEventId` in `description.md` (removed; last version in commit `28a09e9`).

## Context

Each platform needs more than an ID: link state (creating, active, detached), the last hash Iris pushed, the remote update timestamp and platform extras. Nullable columns would repeat these fields four or five times and need a migration for every new platform.

## Decision

Split the model into `Task` (canonical) and `ExternalLink` (one row per task per platform, `@@unique([taskId, platform])` and `@@unique([platform, externalId])`).

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Adding a platform only adds an enum value. Lookups by external ID stay O(1) through the unique index.
