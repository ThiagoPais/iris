---
type: adr
project: iris
number: 004
status: accepted
updated: 2026-11-16
---

# 004. Echo suppression with per-link projected hashes (no time window)

**Supersedes:** "matches `content_hash` within a 5-second window" in `description.md`.

## Context

When Iris pushes to Google Tasks, the Apps Script poller reports the same change back 1–5 minutes later. A 5-second window cannot catch that echo. Worse, an echo of an *older* push can arrive after a newer edit from another platform and revert it.

## Decision

1. After every successful push, Iris stores `ExternalLink.lastSyncedHash = hash(project(platform, task))`.
2. When an inbound change arrives for an echoing platform (Google Tasks, Google Calendar, Habitica) and `hash(project(platform, incoming)) == link.lastSyncedHash`, it is an echo: Iris logs `NOOP_ECHO` and stops.
3. Independently, if the merged canonical state equals the current canonical state, Iris logs `NOOP_DUPLICATE` and stops. This also covers repeated hook deliveries.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Echo detection works however late the echo arrives. A real user edit that reproduces exactly the last pushed state before any other inbound change is treated as an echo, which is harmless because the canonical state already equals it.
