---
type: adr
project: iris
number: 006
status: accepted
updated: 2026-11-16
---

# 006. Field-support projections and partial merge

## Context

Platforms support different fields. Google Tasks has no tags, priority or time of day; Taskwarrior has no free-text notes. A naive full overwrite from a limited platform would erase data.

## Decision

A static **field support matrix** defines, per platform, which canonical fields it can express (see the [PRD field support matrix](../iris-prd.md#canonical-model--field-support-matrix)). `project(platform, task)` returns only those fields, normalized to the platform's precision (for example, a due date reduced to a date for Google Tasks). An inbound change may only set fields its platform supports. For precision-reduced fields, the inbound value is applied only when its projection differs from the canonical value's projection. For example, if Google Tasks reports the same date for a timed due, the time is kept.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Data is never lost by passing through a less expressive platform. Hashes are always computed over projections, so echoes from limited platforms still match.
