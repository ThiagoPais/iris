---
type: adr
project: iris
number: 012
status: accepted
updated: 2026-11-16
---

# 012. Authentication: per-source bearer tokens; origin derived from the credential

**Supersedes:** the client-supplied `origin` header in `description.md`.

## Context

TODO: not recorded at decision time.

## Decision

There are separate secrets for `agent` (the Taskwarrior local agents), `google-script`, `habitica` (URL path secret) and `admin`. The guard compares tokens in constant time and attaches a `Principal { kind }`. Each ingest route accepts exactly one principal kind, and the origin platform is **derived from the route and principal, never from a header or body field**.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

A leaked Google Script token cannot forge agent events. Rotating a secret is an environment change followed by a restart.
