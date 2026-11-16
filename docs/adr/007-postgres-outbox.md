---
type: adr
project: iris
number: 007
status: accepted
updated: 2026-11-16
---

# 007. Durable outbox in PostgreSQL for push targets

**Supersedes:** "native event emitter" dispatch implied in `description.md`.

## Context

Remote APIs fail, rate-limit (Habitica allows 30 req/min) and go down. In-memory events are lost on restart.

## Decision

The ingest transaction also writes `SyncJob` rows, one per push target other than the origin. A partial unique index on `(taskId, target) WHERE status = 'PENDING'` coalesces bursts. An in-process worker claims jobs with `FOR UPDATE SKIP LOCKED`, reads the **current** task state (not a snapshot), pushes it, and retries with exponential backoff (`min(5s·2^attempts, 1h)`, 10 attempts, then `DEAD`). A `RUNNING` job older than 5 minutes goes back to `PENDING`.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Delivery is at-least-once with coalescing, and no extra infrastructure is needed. Remote upserts must therefore be idempotent (ADR-011 alias, ADR-010 extended properties, `PENDING_CREATE` adoption).
