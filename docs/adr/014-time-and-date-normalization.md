---
type: adr
project: iris
number: 014
status: accepted
updated: 2026-11-16
---

# 014. Time and date normalization

## Context

TODO: not recorded at decision time.

## Decision

- All timestamps are `timestamptz` in UTC.
- Due is stored as `dueAt` + `dueAllDay`. An all-day due is stored as `YYYY-MM-DDT00:00:00Z` of the local calendar date.
- `IRIS_TIMEZONE` (IANA, for example `America/Sao_Paulo`) interprets platform dates:
  - A Taskwarrior due at exactly local midnight is treated as all-day.
  - Google Tasks and Habitica dates are always all-day.
  - For timed dues, outbound date-only platforms receive the local date.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

`due:tomorrow` in Taskwarrior round-trips losslessly through Google Tasks.
