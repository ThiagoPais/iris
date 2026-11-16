# Product Requirements Document (PRD) - Project Iris

## 1. Overview

**Project Name:** Iris
**Description:** A personal sync hub and command center. It keeps one canonical list of tasks consistent across Taskwarrior (personal and work PCs), Google Tasks (Android), Google Calendar (scheduled blocks) and Habitica (gamification). It keeps cross-platform ID mappings, resolves edit collisions and never loops.

**Owner / Users:** One user (ADR-016).

### 1.1 Problem

Tasks are captured wherever the user happens to be: the CLI at a desk, the phone on the go, the calendar when planning. Each tool holds a partial, drifting copy. Completing a task in one place does not complete it elsewhere, and Habitica rewards are only earned when the task is re-entered by hand.

### 1.2 Goals

| # | Goal | Success metric |
| --- | --- | --- |
| G1 | A task created on any platform appears on all others | ≥ 99% of creates propagated; p95 latency ≤ 2 min (push targets), ≤ 1 poll interval (agents) |
| G2 | Completion, edits and deletions propagate | Same as G1 |
| G3 | No duplicates and no sync loops | 0 duplicate tasks per 1,000 syncs; 0 unbounded echo chains |
| G4 | No lost edits when a platform or Iris is offline | Every hook event eventually ingested; every push retried until success or `DEAD` |
| G5 | Zero friction on the CLI | Taskwarrior hooks add ≤ 100 ms whether Iris is up or down (no network I/O in the hook); they never fail a command |

### 1.3 Non-goals (MVP)

- Multiple users, sharing or collaboration.
- Habitica habits, dailies and rewards; Google Calendar events outside the dedicated `Iris` calendar.
- Recurring-task semantics: recurrence templates are not synced, their generated instances are.
- Field-level merge of concurrent edits (ADR-005).
- A graphical UI. The command center is a JSON admin API; a UI is a future project.
- Per-platform filters beyond Taskwarrior's `+nosync` tag.

## 2. Core Features (MVP)

- **Taskwarrior sync (bidirectional, multi-device)**
  - `on-add` hook: a newly **created** task is sent to Iris.
  - `on-modify` hook: edits, `done`, `delete`, `start/stop` and annotations are sent to Iris (only changes to supported fields have an effect).
  - Spool-first delivery: the hook queues each event locally and starts a detached flush, so the CLI never waits on the network and nothing is lost while Iris is offline.
  - Local agent (systemd user timer, every 60 s): flushes the spool, pulls remote changes and applies them with hooks disabled. The same task UUID is used on every PC.
  - Bootstrap command: `iris-agent push-all` sends every existing pending task once.
  - `+nosync` tag: excludes a task from sync.
- **Google Tasks sync (bidirectional):** one configured task list. Inbound through Apps Script polling (1 min), outbound through the REST API.
- **Google Calendar sync (bidirectional):** one dedicated calendar. Tasks with a due date appear as events (all-day or a 30-minute block). Moving an event reschedules the task; deleting it unschedules the task.
- **Habitica sync (bidirectional):** tasks become Habitica todos. Completing a task anywhere scores it up (XP and gold), and reopening scores it down. Priority maps to difficulty.
- **Command center (admin API)**
  - List and search tasks with their per-platform link state.
  - Inspect a task's sync history (`SyncLog`).
  - List failed or dead outbound jobs and retry them.
  - Health endpoint reporting DB connectivity and outbox backlog.

## 3. User Stories

| ID | As the user, I want to… | Acceptance |
| --- | --- | --- |
| US-01 | run `task add "Buy milk" due:tomorrow` and see it on my phone | Item appears in the Google Tasks list with tomorrow's date, in Habitica as a todo and in the `Iris` calendar as an all-day event |
| US-02 | tick a task in Google Tasks and have Habitica reward me | Habitica todo is scored up; Taskwarrior on both PCs shows `completed` after their next agent run |
| US-03 | edit a task on my work PC and see it on my personal PC | Personal PC agent imports the change within one poll interval; no hook fires during import |
| US-04 | keep using `task` when Iris is down | Command succeeds with no added network delay; the event is delivered after Iris returns |
| US-05 | drag a calendar event to Friday 14:00 | Task due becomes Friday 14:00 (timed); Google Tasks and Habitica show Friday |
| US-06 | delete a calendar event | Task stays pending with no due; the other platforms clear their dates |
| US-07 | never see a task duplicated because of sync | Echoes and replays are suppressed (`NOOP_ECHO`/`NOOP_DUPLICATE` in logs) |
| US-08 | see what failed to sync and retry it | `GET /v1/sync/jobs?status=DEAD` lists failures; `POST /v1/sync/jobs/:id/retry` re-queues |
| US-09 | have a task created on my phone appear in Taskwarrior | Agents on both PCs import it with the same UUID |
| US-10 | keep private tasks off other platforms | Tasks tagged `+nosync` are not sent by the hook |

## 4. Technical Architecture & Stack

- **Hub:** NestJS 12 (TypeScript, ESM), running in Docker on a VPS behind Caddy with TLS (ADR-015).
- **Database:** PostgreSQL 17 through Prisma ORM (ADR-002). See `iris-ddd.md`.
- **Taskwarrior integration:** Python 3.11+ standard library only. A hook script and a pull agent with a systemd user timer (ADR-008/009).
- **Google integration:** an Apps Script project (inbound) and the `googleapis` REST clients with an OAuth2 refresh token (outbound) (ADR-010).
- **Habitica integration:** REST API v3 and a `taskActivity` webhook (ADR-011).

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

## 5. Canonical Model & Field Support Matrix

Canonical task fields: `title`, `notes`, `status` (`PENDING|COMPLETED|DELETED`), `priority` (`NONE|LOW|MEDIUM|HIGH`), `project`, `tags[]`, `dueAt` + `dueAllDay`, `completedAt`.

| Canonical | Taskwarrior | Google Tasks | Google Calendar | Habitica (todo) |
| --- | --- | --- | --- | --- |
| title | `description` | `title` | `summary` | `text` |
| notes | — | `notes` | `description` | `notes` |
| status | `status` (pending/waiting→PENDING) | `status` + `deleted` | — (event exists only when not DELETED) | `completed` / deletion |
| priority | `priority` H/M/L | — | — | `priority` 0.1/1/1.5/2 |
| project | `project` | — | — | — |
| tags | `tags` | — | — | — |
| due | `due` (full) | `due` (date only) | `start` (full) | `date` (date only) |

"—" means the platform neither sends nor overwrites that field (ADR-006).

## 6. Data Flow & Core Loop

1. **Capture:** A platform emits a change: a hook POST, an Apps Script batch, a Habitica webhook or an agent flush.
2. **Authenticate:** The guard maps the token to a principal. The route fixes the origin platform (ADR-012).
3. **Normalize:** The platform mapper converts the payload into an `InboundChange` (external ID, operation, supported fields, `sourceUpdatedAt`, optional `irisTaskId` hint).
4. **Ingest (one transaction):** advisory lock → resolve or create the link → echo check → staleness check → partial merge → duplicate check → persist `Task` (version + 1), `TaskChange`, `SyncLog`, and a `SyncJob` for every push target except the origin.
5. **Dispatch:** The worker claims due jobs, renders the current task for the target, calls the remote API, and updates `ExternalLink` (ID, state, `lastSyncedHash`). Failures are retried with backoff.
6. **Pull:** Local agents poll the change feed and apply changes with hooks off.
7. **Echo:** When the platform reports Iris's own write back, it matches `lastSyncedHash` and is dropped.

## 7. Non-Functional Requirements

- **Reliability:** at-least-once delivery for both inbound (spool) and outbound (outbox). Every remote write is idempotent.
- **Performance:** ingest p95 ≤ 100 ms server-side, and hook overhead within G5.
- **Security:** TLS everywhere, secrets only in environment variables, tokens compared in constant time, raw payloads logged without secrets. The Habitica secret appears in the URL path, so reverse-proxy access logs must redact `/v1/ingest/habitica/*`.
- **Observability:** structured JSON logs with `taskId`, `platform` and `outcome`. `SyncLog` rows are kept for 90 days (pruned daily).
- **Rate limits:** Habitica ≤ 25 req/min client-side. Google calls use exponential backoff on 429/5xx.
- **Portability:** Agents need only Python 3.11+ and Taskwarrior 2.6+/3.x.

## 8. Phased Delivery

| Phase | Scope | Exit criteria |
| --- | --- | --- |
| 1 — Core Hub | Config, Prisma schema, auth, domain (projection/hash/merge), IngestService, outbox + worker, change feed, admin API, Docker | e2e: ingest → job → fake adapter push → echo suppressed |
| 2 — Taskwarrior | Hook (`on-add`, `on-modify`), spool, agent, installer, TW mapper and endpoints | Two TW data dirs sync through Iris on one machine |
| 3 — External | Google Tasks, Google Calendar, Apps Script, Habitica | US-01…US-08 pass manually against real accounts |
