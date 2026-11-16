# Project: Iris

## Status: Documentation Baseline Established (PRD, DDD, ADR, Specs)

### Overview

Iris is a personal sync hub and command center that keeps tasks consistent across Taskwarrior (personal and work PCs), Google Tasks (Android), Google Calendar and Habitica. It stores one canonical record per task, keeps the cross-platform ID mappings, resolves edit collisions and suppresses sync loops.

### Documents

| Document | Purpose |
| --- | --- |
| [`description.md`](./description.md) | Original brief (PRD/ADD draft). Superseded where the ADRs say so. |
| [`iris-prd.md`](./iris-prd.md) | Product requirements: goals, features, user stories, field matrix, phases. |
| [`iris-ddd.md`](./iris-ddd.md) | Database design: ERD, authoritative Prisma schema, constraints, raw SQL. |
| [`iris-adr.md`](./iris-adr.md) | Architecture decision records (ADR-001…016). |
| [`../spec/`](../spec/README.md) | Execution specs for AI agents, one per work package. |

### Core Features (MVP)

- **Taskwarrior sync:** `on-add` and `on-modify` hooks trigger on task **creation** and **modification**. Spool-first delivery (no network I/O in the hook) plus a pull agent keep personal and work PCs in sync under the same task UUIDs.
- **Google Tasks & Calendar sync:** Apps Script pushes inbound changes every minute; Iris writes outbound through the REST APIs. Tasks with a due date appear as events on a dedicated `Iris` calendar.
- **Habitica sync:** tasks become todos, and completing a task anywhere scores it up for XP and gold.
- **Loop-free, conflict-aware core:** echo suppression through per-link projected hashes, last-writer-wins on source timestamps, and partial merge by field support.
- **Command center:** an admin JSON API to inspect tasks, links and sync history, and to retry failed jobs.

### Technical Stack

- **Hub:** NestJS 12 (TypeScript, ESM), Vitest, oxlint, Prettier.
- **Database:** PostgreSQL 17 through Prisma ORM.
- **Local agents:** Python 3.11+ (standard library only) with a systemd user timer.
- **Google:** Apps Script (inbound) and the `googleapis` Tasks v1 / Calendar v3 clients with an OAuth2 refresh token (outbound).
- **Habitica:** REST API v3 and a `taskActivity` webhook.
- **Deployment:** Docker Compose (hub + Postgres) on a VPS behind Caddy with TLS. Public HTTPS is required for the Google and Habitica inbound calls.

### Database & Core Architecture

- **Canonical store:** `tasks` is the source of truth; `external_links` maps each task to one identifier per platform (ADR-003).
- **Transactional ingest:** each inbound change is applied in one transaction that writes the task, a `task_changes` feed row, a `sync_logs` audit row and one `sync_jobs` outbox row per push target.
- **Outbox worker:** claims jobs with `FOR UPDATE SKIP LOCKED` and retries with exponential backoff until `DEAD` (ADR-007).
- **Change feed:** a monotonic `task_changes.id` serves as the cursor for local agents (ADR-008).
- **Idempotency:** Habitica aliases (`iris-<taskId>`), Calendar `extendedProperties.private.irisTaskId`, and `PENDING_CREATE` link adoption for Google Tasks.

### Security Blueprint

- Separate bearer tokens per source (`agent`, `google-script`, `admin`), plus a URL path secret for the Habitica webhook.
- The origin platform is derived from the route and credential, never from a client-supplied header (ADR-012).
- TLS terminates at Caddy. Secrets exist only in environment variables, and access logs for the Habitica route are redacted.

### Progress Log

- **[2026-09-16] Project Brief**
  - Initial PRD/ADD draft (`description.md`) covering the topology, anti-looping strategy, first schema and roadmap.
- **[2026-09-30] Repository & Documentation Baseline**
  - NestJS 12 ESM scaffold created (Vitest, oxlint, Prettier).
  - `description.md` updated: Taskwarrior sync is triggered by task **creation** (`on-add`) as well as modification (`on-modify`).
  - PRD, DDD and ADR (001–016) written. The schema was refactored to `Task` + `ExternalLink` + `TaskChange` + `SyncJob` + `SyncLog` + `AgentDevice`.
  - AI-agent execution specs written in `spec/`.

### Next Steps (Action Plan)

#### Phase 1: Core Hub (specs 01–05)

- Environment config, Docker Compose Postgres, Prisma schema and migrations (including raw SQL).
- Authentication guard, domain layer (projection, hashing, merge, time normalization).
- `IngestService`, outbox and `SyncWorker`, change feed, admin API, health endpoint.

#### Phase 2: Taskwarrior (spec 06)

- `iris_hook.py` (`on-add`/`on-modify`), spool, `iris-agent` (flush/pull/push-all), installer and systemd timer.
- Taskwarrior mapper, ingest endpoint and feed endpoint.

#### Phase 3: External Integrations (specs 07–09)

- Google OAuth CLI, Google Tasks and Calendar adapters, Apps Script project.
- Habitica client (throttled), adapter and webhook.

#### Phase 4: Deployment (spec 10)

- Production Dockerfile, Caddy, backups and maintenance jobs.
