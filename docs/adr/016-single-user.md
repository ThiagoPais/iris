---
type: adr
project: iris
number: 016
status: accepted
updated: 2026-11-16
---

# 016. Single-user system

## Context

TODO: not recorded at decision time.

## Decision

One owner, with no user or tenant model. Credentials for external platforms live in environment variables.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

The schema and authentication are much simpler. Multi-user support would need a new ADR and a migration adding an `ownerId` to every table.
