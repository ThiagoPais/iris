---
type: adr
project: iris
number: 001
status: accepted
updated: 2026-11-16
---

# 001. Hub-and-spoke with a canonical task store

## Context

Four task systems must stay in sync. With point-to-point sync, N platforms need N×(N−1) translators, and there is no single source of truth for conflicts.

## Decision

Iris keeps one **canonical task** per real-world task in PostgreSQL. Every platform talks only to Iris. Inbound changes are normalized into the canonical model, and outbound changes are rendered from it.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

- (+) N translators instead of N², a single place for conflict rules, and a full audit trail.
- (−) Iris is a single point of failure. Mitigated by local spooling (ADR-009) and outbox retries (ADR-007).
