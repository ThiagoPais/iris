---
type: roadmap
project: iris
status: active
updated: 2026-11-16
---

# Iris: Roadmap

## Current phase

**Documentation Baseline Established (PRD, DDD, ADR, Specs).** Implementation has not started; spec 01 (foundation) is next.

## Phases

| Phase | Scope | Exit criteria |
| --- | --- | --- |
| 1 — Core Hub | Config, Prisma schema, auth, domain (projection/hash/merge), IngestService, outbox + worker, change feed, admin API, Docker | e2e: ingest → job → fake adapter push → echo suppressed |
| 2 — Taskwarrior | Hook (`on-add`, `on-modify`), spool, agent, installer, TW mapper and endpoints | Two TW data dirs sync through Iris on one machine |
| 3 — External | Google Tasks, Google Calendar, Apps Script, Habitica | US-01…US-08 pass manually against real accounts |

<!-- TODO: phase 4 (deployment, spec 10) has no exit criteria yet. -->

## Next steps

### Phase 1: Core Hub (specs 01–05)

- Environment config, Docker Compose Postgres, Prisma schema and migrations (including raw SQL).
- Authentication guard, domain layer (projection, hashing, merge, time normalization).
- `IngestService`, outbox and `SyncWorker`, change feed, admin API, health endpoint.

### Phase 2: Taskwarrior (spec 06)

- `iris_hook.py` (`on-add`/`on-modify`), spool, `iris-agent` (flush/pull/push-all), installer and systemd timer.
- Taskwarrior mapper, ingest endpoint and feed endpoint.

### Phase 3: External Integrations (specs 07–09)

- Google OAuth CLI, Google Tasks and Calendar adapters, Apps Script project.
- Habitica client (throttled), adapter and webhook.

### Phase 4: Deployment (spec 10)

- Production Dockerfile, Caddy, backups and maintenance jobs.

## Log

- **[2026-11-16] Scope and docs**
  - Calcurse dropped from scope (former ADR-017 and spec 10 removed; deployment is now spec 10).
  - Documentation migrated to the PanWare conventions (overview, PRD, DBDD, roadmap, `adr/`).
- **[2026-09-30] Repository & Documentation Baseline**
  - NestJS 12 ESM scaffold created (Vitest, oxlint, Prettier).
  - `description.md` updated: Taskwarrior sync is triggered by task **creation** (`on-add`) as well as modification (`on-modify`).
  - PRD, DDD and ADR (001–016) written. The schema was refactored to `Task` + `ExternalLink` + `TaskChange` + `SyncJob` + `SyncLog` + `AgentDevice`.
  - AI-agent execution specs written in `spec/`.
- **[2026-09-16] Project Brief**
  - Initial PRD/ADD draft (`description.md`) covering the topology, anti-looping strategy, first schema and roadmap.
