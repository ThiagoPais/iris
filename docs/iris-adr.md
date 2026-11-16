# Architecture Decision Records (ADR) - Project Iris

This file collects the architecture decisions for Iris. Each record is immutable once `Accepted`. To change a decision, add a new record that supersedes it and set the old one's status to `Superseded by ADR-XXX`.

`docs/description.md` is the original brief. Where a record below differs from it, the record wins, and the record says so under **Supersedes**.

**Status legend:** `Proposed` · `Accepted` · `Superseded` · `Deprecated`

| ADR | Title | Status |
| --- | --- | --- |
| 001 | Hub-and-spoke with a canonical task store | Accepted |
| 002 | NestJS 12 (ESM) + PostgreSQL + Prisma | Accepted |
| 003 | Per-platform link table instead of nullable ID columns | Accepted |
| 004 | Echo suppression with per-link projected hashes (no time window) | Accepted |
| 005 | Conflict resolution: record-level last-writer-wins on source timestamps | Accepted |
| 006 | Field-support projections and partial merge | Accepted |
| 007 | Durable outbox in PostgreSQL for push targets | Accepted |
| 008 | Pull-based change feed for local agents (Taskwarrior) | Accepted |
| 009 | Taskwarrior hooks: `on-add` + `on-modify`, spool on failure | Accepted |
| 010 | Google Tasks & Calendar: Apps Script inbound, REST API outbound | Accepted |
| 011 | Habitica: todos only, webhooks inbound, alias-based identity | Accepted |
| 012 | Authentication: per-source bearer tokens; origin derived from the credential | Accepted |
| 013 | Deletion semantics: soft delete; calendar deletion unschedules | Accepted |
| 014 | Time and date normalization | Accepted |
| 015 | Deployment: single container + Postgres, reachable over public HTTPS | Accepted |
| 016 | Single-user system | Accepted |

---

## ADR-001: Hub-and-spoke with a canonical task store

- **Status:** Accepted (2026-09-30)
- **Context:** Four task systems must stay in sync. With point-to-point sync, N platforms need N×(N−1) translators, and there is no single source of truth for conflicts.
- **Decision:** Iris keeps one **canonical task** per real-world task in PostgreSQL. Every platform talks only to Iris. Inbound changes are normalized into the canonical model, and outbound changes are rendered from it.
- **Consequences:**
  - (+) N translators instead of N², a single place for conflict rules, and a full audit trail.
  - (−) Iris is a single point of failure. Mitigated by local spooling (ADR-009) and outbox retries (ADR-007).

## ADR-002: NestJS 12 (ESM) + PostgreSQL + Prisma

- **Status:** Accepted (2026-09-30)
- **Context:** The repository is already scaffolded with NestJS 12 as ESM (`"type": "module"`, `.js` import suffixes), Vitest, oxlint and Prettier.
- **Decision:** Keep that scaffold. Use PostgreSQL 17 and the latest Prisma major version. Use `$queryRaw` only where Prisma cannot express something: `FOR UPDATE SKIP LOCKED`, advisory locks and partial unique indexes. Validate environment variables with `zod` at boot.
- **Consequences:** ACID transactions keep ingest and outbox writes atomic, and no Redis or broker is needed. The `@nestjs/observe` wiring in the scaffold carries placeholder credentials, so it is registered only when `OBSERVE_APP_KEY` is set.

## ADR-003: Per-platform link table instead of nullable ID columns

- **Status:** Accepted (2026-09-30)
- **Supersedes:** `TaskMapping.taskwarriorUuid/googleTaskId/habiticaId/googleEventId` in `description.md`.
- **Context:** Each platform needs more than an ID: link state (creating, active, detached), the last hash Iris pushed, the remote update timestamp and platform extras. Nullable columns would repeat these fields four or five times and need a migration for every new platform.
- **Decision:** Split the model into `Task` (canonical) and `ExternalLink` (one row per task per platform, `@@unique([taskId, platform])` and `@@unique([platform, externalId])`).
- **Consequences:** Adding a platform only adds an enum value. Lookups by external ID stay O(1) through the unique index.

## ADR-004: Echo suppression with per-link projected hashes (no time window)

- **Status:** Accepted (2026-09-30)
- **Supersedes:** "matches `content_hash` within a 5-second window" in `description.md`.
- **Context:** When Iris pushes to Google Tasks, the Apps Script poller reports the same change back 1–5 minutes later. A 5-second window cannot catch that echo. Worse, an echo of an *older* push can arrive after a newer edit from another platform and revert it.
- **Decision:**
  1. After every successful push, Iris stores `ExternalLink.lastSyncedHash = hash(project(platform, task))`.
  2. When an inbound change arrives for an echoing platform (Google Tasks, Google Calendar, Habitica) and `hash(project(platform, incoming)) == link.lastSyncedHash`, it is an echo: Iris logs `NOOP_ECHO` and stops.
  3. Independently, if the merged canonical state equals the current canonical state, Iris logs `NOOP_DUPLICATE` and stops. This also covers repeated hook deliveries.
- **Consequences:** Echo detection works however late the echo arrives. A real user edit that reproduces exactly the last pushed state before any other inbound change is treated as an echo, which is harmless because the canonical state already equals it.

## ADR-005: Conflict resolution: record-level last-writer-wins on source timestamps

- **Status:** Accepted (2026-09-30)
- **Context:** The same task can be edited on two platforms between syncs.
- **Decision:** Every inbound change carries `sourceUpdatedAt`, the platform's own modification time (Taskwarrior `modified`, Google `updated`, Habitica `updatedAt`). Iris stores the winning edit's time as `Task.lastEditAt`. An inbound change with `sourceUpdatedAt < lastEditAt − 2s` (skew tolerance) is `REJECTED_STALE`. Iris does not apply it and enqueues a push back to the origin, when the origin is a push target, so that platform converges to canonical.
- **Consequences:** The rule is simple and deterministic. Edits to different fields on two platforms within one sync interval lose the older edit; field-level merge is a possible future ADR. Clock skew beyond 2 s between PCs can misorder edits.

## ADR-006: Field-support projections and partial merge

- **Status:** Accepted (2026-09-30)
- **Context:** Platforms support different fields. Google Tasks has no tags, priority or time of day; Taskwarrior has no free-text notes. A naive full overwrite from a limited platform would erase data.
- **Decision:** A static **field support matrix** defines, per platform, which canonical fields it can express (see PRD §5). `project(platform, task)` returns only those fields, normalized to the platform's precision (for example, a due date reduced to a date for Google Tasks). An inbound change may only set fields its platform supports. For precision-reduced fields, the inbound value is applied only when its projection differs from the canonical value's projection. For example, if Google Tasks reports the same date for a timed due, the time is kept.
- **Consequences:** Data is never lost by passing through a less expressive platform. Hashes are always computed over projections, so echoes from limited platforms still match.

## ADR-007: Durable outbox in PostgreSQL for push targets

- **Status:** Accepted (2026-09-30)
- **Supersedes:** "native event emitter" dispatch implied in `description.md`.
- **Context:** Remote APIs fail, rate-limit (Habitica allows 30 req/min) and go down. In-memory events are lost on restart.
- **Decision:** The ingest transaction also writes `SyncJob` rows, one per push target other than the origin. A partial unique index on `(taskId, target) WHERE status = 'PENDING'` coalesces bursts. An in-process worker claims jobs with `FOR UPDATE SKIP LOCKED`, reads the **current** task state (not a snapshot), pushes it, and retries with exponential backoff (`min(5s·2^attempts, 1h)`, 10 attempts, then `DEAD`). A `RUNNING` job older than 5 minutes goes back to `PENDING`.
- **Consequences:** Delivery is at-least-once with coalescing, and no extra infrastructure is needed. Remote upserts must therefore be idempotent (ADR-011 alias, ADR-010 extended properties, `PENDING_CREATE` adoption).

## ADR-008: Pull-based change feed for local agents (Taskwarrior)

- **Status:** Accepted (2026-09-30)
- **Context:** The hub cannot call into a laptop behind NAT, and Taskwarrior has no remote API. Two PCs (personal and work) each hold a local Taskwarrior database.
- **Decision:** Every canonical state change appends a `TaskChange` row with a monotonic `BigInt` id. A local agent on each device polls `GET /v1/agents/{platform}/changes?deviceId&since` and receives the current state of every task that changed after its cursor, excluding changes whose `originDevice` is the caller. The agent applies them locally with hooks disabled and keeps the cursor on disk. Taskwarrior tasks keep the **same UUID on every device**. When the feed serves a task that has no link for the agent's platform, Iris creates one on the spot (`ACTIVE`, fresh UUID v4 as `externalId`) inside the feed transaction.
- **Consequences:** No inbound connectivity to PCs is required, and a device that was offline catches up when it returns. Latency equals the agent poll interval (default 60 s, systemd user timer).

## ADR-009: Taskwarrior hooks: `on-add` + `on-modify`, spool on failure

- **Status:** Accepted (2026-09-30)
- **Context:** Iris must react both when a task is **created** and when it is **modified** (including `done` and `delete`). Hooks run synchronously inside the `task` CLI.
- **Decision:**
  - One Python 3 (stdlib-only) script, `iris_hook.py`, symlinked as `~/.task/hooks/on-add.iris` and `~/.task/hooks/on-modify.iris`. The mode comes from `basename(argv[0])`.
  - The hook always echoes the required JSON on stdout and exits `0`, and it never blocks the task.
  - **Spool-first:** the hook never touches the network. It appends the event to `~/.local/state/iris/spool.jsonl` (under an `fcntl` lock) and starts a **detached** `iris_agent.py flush` (new session, all stdio on `/dev/null`), then returns. The flush POSTs spooled events in order and removes only the ones Iris acknowledged. The timer-driven agent flushes again before every pull, so events survive Iris being offline.
  - Tasks tagged `+nosync` and recurrence templates (`status: recurring`) are not sent.
  - The agent applies remote changes with `task rc.hooks=off rc.confirmation=off import`, so applying them can never re-trigger the hooks.
- **Consequences:** Hook cost is Python startup plus one file append (≤ 100 ms) whether Iris is up or down. Events are never lost, and Iris writes to Taskwarrior do not loop back. Latency to Iris is the time for the detached flush to run (well under 1 s when Iris is up).

## ADR-010: Google Tasks & Calendar: Apps Script inbound, REST API outbound

- **Status:** Accepted (2026-09-30)
- **Context:** Google Tasks has no push notifications. Calendar push channels need a verified domain and renewal.
- **Decision:**
  - **Inbound:** one Apps Script project with a 1-minute time-driven trigger. For Tasks, it lists `updatedMin = lastRun − 60s` with `showDeleted/showHidden/showCompleted`. For Calendar, it uses `Events.list` with a `syncToken` on **one dedicated calendar** (`Iris`). Changes are POSTed in batches to `/v1/ingest/google-tasks` and `/v1/ingest/google-calendar`.
  - **Outbound:** Iris calls the Tasks v1 and Calendar v3 REST APIs with an OAuth2 refresh token (scopes `tasks`, `calendar.events`). A one-time loopback CLI (`npm run google:auth`) obtains the token. Iris syncs **one configured task list** (`GOOGLE_TASKLIST_ID`) and **one calendar** (`GOOGLE_CALENDAR_ID`).
  - **Identity:** calendar events carry `extendedProperties.private.irisTaskId`. Google Tasks cannot carry a client ID, so a link sits in `PENDING_CREATE` during insert, and an inbound item with an unknown ID whose projected hash matches a `PENDING_CREATE` link is adopted instead of creating a duplicate.
  - **Calendar projection:** only non-deleted tasks with a due are events. An all-day due becomes an all-day event; a timed due becomes a `[due, due + GOOGLE_CALENDAR_EVENT_MINUTES]` event (default 30).
- **Consequences:** Inbound latency is about 1–2 minutes, and personal calendars are never imported.

## ADR-011: Habitica: todos only, webhooks inbound, alias-based identity

- **Status:** Accepted (2026-09-30)
- **Context:** Habitica has habits, dailies, todos and rewards; only todos map to one-shot tasks. It supports user webhooks (`taskActivity`), which are not signed.
- **Decision:**
  - Iris syncs only `type: "todo"`.
  - **Outbound:** `POST /api/v3/tasks/user` with `alias: "iris-<taskId>"`, `PUT /api/v3/tasks/{id}` for fields, `POST /api/v3/tasks/{id}/score/up` on completion, `score/down` on reopen, and `DELETE` on deletion. Headers are `x-api-user`, `x-api-key` and `x-client: <userId>-iris`. The client throttles to ≤ 25 req/min.
  - **Inbound:** a `taskActivity` webhook (created/updated/deleted/scored) to `POST /v1/ingest/habitica/{HABITICA_WEBHOOK_SECRET}`.
  - **Priority mapping:** `NONE↔0.1`, `LOW↔1`, `MEDIUM↔1.5`, `HIGH↔2`.
- **Consequences:** Completing a task anywhere grants Habitica rewards, and the alias makes webhook echoes resolvable before the link commits.

## ADR-012: Authentication: per-source bearer tokens; origin derived from the credential

- **Status:** Accepted (2026-09-30)
- **Supersedes:** the client-supplied `origin` header in `description.md`.
- **Decision:** There are separate secrets for `agent` (the Taskwarrior local agents), `google-script`, `habitica` (URL path secret) and `admin`. The guard compares tokens in constant time and attaches a `Principal { kind }`. Each ingest route accepts exactly one principal kind, and the origin platform is **derived from the route and principal, never from a header or body field**.
- **Consequences:** A leaked Google Script token cannot forge agent events. Rotating a secret is an environment change followed by a restart.

## ADR-013: Deletion semantics: soft delete; calendar deletion unschedules

- **Status:** Accepted (2026-09-30)
- **Decision:**
  - Canonical tasks are never hard-deleted. `status = DELETED` propagates as a remote delete to push targets and as `status: deleted` to agents. Links become `DETACHED`.
  - A `DELETED → PENDING` revival (for example, Taskwarrior undo) re-creates remote items through new creates.
  - **Exception:** deleting or cancelling a Google Calendar event **clears the task's due date** instead of deleting the task, because removing a calendar block usually means "unschedule", not "drop".
- **Consequences:** Deletions can be recovered and the audit trail stays complete. Storage grows slowly, and a purge job is out of scope.

## ADR-014: Time and date normalization

- **Status:** Accepted (2026-09-30)
- **Decision:**
  - All timestamps are `timestamptz` in UTC.
  - Due is stored as `dueAt` + `dueAllDay`. An all-day due is stored as `YYYY-MM-DDT00:00:00Z` of the local calendar date.
  - `IRIS_TIMEZONE` (IANA, for example `America/Sao_Paulo`) interprets platform dates:
    - A Taskwarrior due at exactly local midnight is treated as all-day.
    - Google Tasks and Habitica dates are always all-day.
    - For timed dues, outbound date-only platforms receive the local date.
- **Consequences:** `due:tomorrow` in Taskwarrior round-trips losslessly through Google Tasks.

## ADR-015: Deployment: single container + Postgres, reachable over public HTTPS

- **Status:** Accepted (2026-09-30)
- **Context:** Google Apps Script and Habitica webhooks must reach Iris from the internet.
- **Decision:** Deploy with Docker Compose (`iris-hub` + `postgres:17`) on a VPS behind a TLS reverse proxy (Caddy). A home host exposed through Cloudflare Tunnel or Tailscale Funnel is an equivalent alternative. Migrations run on container start (`prisma migrate deploy`).
- **Consequences:** A purely local, unexposed daemon is not supported while Google or Habitica inbound is enabled.

## ADR-016: Single-user system

- **Status:** Accepted (2026-09-30)
- **Decision:** One owner, with no user or tenant model. Credentials for external platforms live in environment variables.
- **Consequences:** The schema and authentication are much simpler. Multi-user support would need a new ADR and a migration adding an `ownerId` to every table.
