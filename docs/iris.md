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

## Documentation map

| Document | Purpose |
| --- | --- |
| [`description.md`](./description.md) | Original brief (PRD/ADD draft). Superseded where the ADRs say so. |
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
