---
type: adr-index
project: iris
status: active
updated: 2026-11-16
---

# Iris: Architectural Decision Records

This file collects the architecture decisions for Iris. Each record is immutable once `Accepted`. To change a decision, add a new record that supersedes it and set the old one's status to `Superseded by ADR-XXX`.

`docs/description.md` is the original brief. Where a record below differs from it, the record wins, and the record says so under **Supersedes**.

**Status legend:** `Proposed` · `Accepted` · `Superseded` · `Deprecated`

| # | Title | Status | Date |
|---|---|---|---|
| [001](001-hub-and-spoke-canonical-store.md) | Hub-and-spoke with a canonical task store | accepted | 2026-09-30 |
| [002](002-nestjs-postgres-prisma.md) | NestJS 12 (ESM) + PostgreSQL + Prisma | accepted | 2026-09-30 |
| [003](003-per-platform-link-table.md) | Per-platform link table instead of nullable ID columns | accepted | 2026-09-30 |
| [004](004-echo-suppression-projected-hashes.md) | Echo suppression with per-link projected hashes (no time window) | accepted | 2026-09-30 |
| [005](005-last-writer-wins-source-timestamps.md) | Conflict resolution: record-level last-writer-wins on source timestamps | accepted | 2026-09-30 |
| [006](006-field-support-projections.md) | Field-support projections and partial merge | accepted | 2026-09-30 |
| [007](007-postgres-outbox.md) | Durable outbox in PostgreSQL for push targets | accepted | 2026-09-30 |
| [008](008-pull-change-feed-for-agents.md) | Pull-based change feed for local agents (Taskwarrior) | accepted | 2026-09-30 |
| [009](009-taskwarrior-hooks-spool.md) | Taskwarrior hooks: `on-add` + `on-modify`, spool on failure | accepted | 2026-09-30 |
| [010](010-google-apps-script-inbound-rest-outbound.md) | Google Tasks & Calendar: Apps Script inbound, REST API outbound | accepted | 2026-09-30 |
| [011](011-habitica-todos-webhooks-aliases.md) | Habitica: todos only, webhooks inbound, alias-based identity | accepted | 2026-09-30 |
| [012](012-per-source-bearer-tokens.md) | Authentication: per-source bearer tokens; origin derived from the credential | accepted | 2026-09-30 |
| [013](013-soft-delete-semantics.md) | Deletion semantics: soft delete; calendar deletion unschedules | accepted | 2026-09-30 |
| [014](014-time-and-date-normalization.md) | Time and date normalization | accepted | 2026-09-30 |
| [015](015-docker-compose-public-https.md) | Deployment: single container + Postgres, reachable over public HTTPS | accepted | 2026-09-30 |
| [016](016-single-user.md) | Single-user system | accepted | 2026-09-30 |
