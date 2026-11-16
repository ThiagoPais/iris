---
type: adr
project: iris
number: 013
status: accepted
updated: 2026-11-16
---

# 013. Deletion semantics: soft delete; calendar deletion unschedules

## Context

TODO: not recorded at decision time.

## Decision

- Canonical tasks are never hard-deleted. `status = DELETED` propagates as a remote delete to push targets and as `status: deleted` to agents. Links become `DETACHED`.
- A `DELETED → PENDING` revival (for example, Taskwarrior undo) re-creates remote items through new creates.
- **Exception:** deleting or cancelling a Google Calendar event **clears the task's due date** instead of deleting the task, because removing a calendar block usually means "unschedule", not "drop".

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Deletions can be recovered and the audit trail stays complete. Storage grows slowly, and a purge job is out of scope.
