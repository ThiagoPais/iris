---
type: adr
project: iris
number: 008
status: accepted
updated: 2026-11-16
---

# 008. Pull-based change feed for local agents (Taskwarrior)

## Context

The hub cannot call into a laptop behind NAT, and Taskwarrior has no remote API. Two PCs (personal and work) each hold a local Taskwarrior database.

## Decision

Every canonical state change appends a `TaskChange` row with a monotonic `BigInt` id. A local agent on each device polls `GET /v1/agents/{platform}/changes?deviceId&since` and receives the current state of every task that changed after its cursor, excluding changes whose `originDevice` is the caller. The agent applies them locally with hooks disabled and keeps the cursor on disk. Taskwarrior tasks keep the **same UUID on every device**. When the feed serves a task that has no link for the agent's platform, Iris creates one on the spot (`ACTIVE`, fresh UUID v4 as `externalId`) inside the feed transaction.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

No inbound connectivity to PCs is required, and a device that was offline catches up when it returns. Latency equals the agent poll interval (default 60 s, systemd user timer).
