---
type: overview
project: iris
status: active
updated: 2026-11-16
---

# Iris

## Summary

Iris is a personal sync hub and command center that keeps tasks consistent across Taskwarrior (personal and work PCs), Google Tasks (Android), Google Calendar and Habitica. It stores one canonical record per task, keeps the cross-platform ID mappings, resolves edit collisions and suppresses sync loops.

## Goals

TODO (docs-author): 3–5 outcomes. The business detail lives in the [PRD goals](./iris-prd.md).

## Repositories

| Repository | Purpose | URL |
|---|---|---|
| iris-hub | Hub (NestJS), local agents, Apps Script, docs | TODO: no remote yet |

## Stack

- **Hub:** NestJS 12 (TypeScript, ESM), Vitest, oxlint, Prettier.
- **Database:** PostgreSQL 17 through Prisma ORM.
- **Local agents:** Python 3.11+ (standard library only) with a systemd user timer.
- **Google:** Apps Script (inbound) and the `googleapis` Tasks v1 / Calendar v3 clients with an OAuth2 refresh token (outbound).
- **Habitica:** REST API v3 and a `taskActivity` webhook.
- **Deployment:** Docker Compose (hub + Postgres) on a VPS behind Caddy with TLS. Public HTTPS is required for the Google and Habitica inbound calls.

<!-- TODO (docs-author): the bullets below came from the PRD and repeat the list above; keep one. -->

- **Hub:** NestJS 12 (TypeScript, ESM), running in Docker on a VPS behind Caddy with TLS (ADR-015).
- **Database:** PostgreSQL 17 through Prisma ORM (ADR-002). See [`iris-dbdd.md`](./iris-dbdd.md).
- **Taskwarrior integration:** Python 3.11+ standard library only. A hook script and a pull agent with a systemd user timer (ADR-008/009).
- **Google integration:** an Apps Script project (inbound) and the `googleapis` REST clients with an OAuth2 refresh token (outbound) (ADR-010).
- **Habitica integration:** REST API v3 and a `taskActivity` webhook (ADR-011).

## Architecture

- **Canonical store:** `tasks` is the source of truth; `external_links` maps each task to one identifier per platform (ADR-003).
- **Transactional ingest:** each inbound change is applied in one transaction that writes the task, a `task_changes` feed row, a `sync_logs` audit row and one `sync_jobs` outbox row per push target.
- **Outbox worker:** claims jobs with `FOR UPDATE SKIP LOCKED` and retries with exponential backoff until `DEAD` (ADR-007).
- **Change feed:** a monotonic `task_changes.id` serves as the cursor for local agents (ADR-008).
- **Idempotency:** Habitica aliases (`iris-<taskId>`), Calendar `extendedProperties.private.irisTaskId`, and `PENDING_CREATE` link adoption for Google Tasks.

### Security blueprint

- Separate bearer tokens per source (`agent`, `google-script`, `admin`), plus a URL path secret for the Habitica webhook.
- The origin platform is derived from the route and credential, never from a client-supplied header (ADR-012).
- TLS terminates at Caddy. Secrets exist only in environment variables, and access logs for the Habitica route are redacted.

### Topology

```text
 PC (personal / work)                                   Internet
┌───────────────────────────┐                 ┌───────────────────────────┐
│ task CLI                  │                 │ Google Apps Script (1 min)│
│  ├ on-add.iris  ─┐        │ POST /v1/ingest │  Tasks + Calendar poll    │
│  └ on-modify.iris┤───────►│────────┐        └─────────────┬─────────────┘
│  spool.jsonl ◄───┘(fail)  │        │                      │ POST /v1/ingest/google-*
│ iris-agent (timer 60s)    │        ▼                      ▼
│  flush spool, pull feed ◄─┼── GET /v1/agents/taskwarrior/changes
│  task rc.hooks=off import │   ┌────────────────────────────────────────┐
└───────────────────────────┘   │ IRIS HUB                               │
                                │  AuthGuard → IngestService (tx):       │
 Habitica ── webhook ──────────►│   lock → resolve link → echo? → LWW →  │
   ▲                            │   merge → persist Task, TaskChange,    │
   │                            │   SyncLog, SyncJob(s)                  │
   │ REST v3                    │  SyncWorker: claim jobs SKIP LOCKED →  │
   └────────────────────────────│   adapter.push → link.lastSyncedHash   │
 Google Tasks / Calendar ◄──────│  PostgreSQL                            │
                    REST        └────────────────────────────────────────┘
```

### Data flow

1. **Capture:** A platform emits a change: a hook POST, an Apps Script batch, a Habitica webhook or an agent flush.
2. **Authenticate:** The guard maps the token to a principal. The route fixes the origin platform (ADR-012).
3. **Normalize:** The platform mapper converts the payload into an `InboundChange` (external ID, operation, supported fields, `sourceUpdatedAt`, optional `irisTaskId` hint).
4. **Ingest (one transaction):** advisory lock → resolve or create the link → echo check → staleness check → partial merge → duplicate check → persist `Task` (version + 1), `TaskChange`, `SyncLog`, and a `SyncJob` for every push target except the origin.
5. **Dispatch:** The worker claims due jobs, renders the current task for the target, calls the remote API, and updates `ExternalLink` (ID, state, `lastSyncedHash`). Failures are retried with backoff.
6. **Pull:** Local agents poll the change feed and apply changes with hooks off.
7. **Echo:** When the platform reports Iris's own write back, it matches `lastSyncedHash` and is dropped.

## Documentation map

| Document | Purpose |
| --- | --- |
| [`iris-prd.md`](./iris-prd.md) | Product requirements: goals, features, user stories, field matrix, phases. |
| [`iris-dbdd.md`](./iris-dbdd.md) | Database design: ERD, authoritative Prisma schema, constraints, raw SQL. |
| [`adr/`](./adr/README.md) | Architecture decision records (ADR-001…016). |
| [`../spec/`](../spec/README.md) | Execution specs for AI agents, one per work package. |
| [`iris-roadmap.md`](./iris-roadmap.md) | Phases, status, next steps and log. |

## Current state

Documentation Baseline Established (PRD, DDD, ADR, Specs). Implementation has not started. See the [roadmap](./iris-roadmap.md).

## Core features (MVP)

<!-- TODO (docs-author): duplicates the PRD's functional requirements; keep one owner. -->

- **Taskwarrior sync:** `on-add` and `on-modify` hooks trigger on task **creation** and **modification**. Spool-first delivery (no network I/O in the hook) plus a pull agent keep personal and work PCs in sync under the same task UUIDs.
- **Google Tasks & Calendar sync:** Apps Script pushes inbound changes every minute; Iris writes outbound through the REST APIs. Tasks with a due date appear as events on a dedicated `Iris` calendar.
- **Habitica sync:** tasks become todos, and completing a task anywhere scores it up for XP and gold.
- **Loop-free, conflict-aware core:** echo suppression through per-link projected hashes, last-writer-wins on source timestamps, and partial merge by field support.
- **Command center:** an admin JSON API to inspect tasks, links and sync history, and to retry failed jobs.
