# Product Requirements & Architecture Document (PRD/ADD) - Project Iris

## Overview & Problem Statement

### Context

Personal productivity tracking is currently fragmented across distinct environments and devices:

* **Desktop/Workstation (Linux/CLI):** Taskwarrior for low-friction local task tracking.
* **Mobile (Android):** Google Tasks for portable checklist updates.
* **Time/Event Tracking:** Google Calendar for scheduled blocks and calendar sync.
* **Gamification/Habits:** Habitica for RPG-style habit formation and task-completion rewards.

### Objective

Build Iris, a centralized sync hub and command center that orchestrates bidirectional synchronization, maintains strict cross-platform ID mapping, resolves edit collisions, and prevents infinite webhook/sync recursion.

## Technical Stack

| Layer | Technology | Rationale |
| --- | --- | --- |
| Runtime & Backend | NestJS (TypeScript) | Modular architecture, native event emitter, robust HTTP/REST routing. |
| Database & ORM | PostgreSQL + Prisma ORM | ACID transactions, composite constraints, JSON metadata storage. |
| Desktop Hook Agents | Python 3 / Bash | Native integration with Taskwarrior hook system (on-modify, on-add). |
| Mobile Capture Script | Google Apps Script | Polling/triggering execution on Google Tasks/Calendar updates. |
| Deployment Target | Docker (Local daemon or VPS) | Containerized service listening to local/remote webhook payloads. |

## System Architecture & Sync Flow

### Topology (Hub-and-Spoke)

```text
[Taskwarrior Hooks: on-add + on-modify] (Personal/Work PC)
         │  (HTTP POST /hooks/taskwarrior)
         ▼
 ┌────────────────────────────────────────────────────────┐
 │                   IRIS HUB (NestJS)                    │
 │                                                        │
 │   ┌─────────────────┐       ┌──────────────────────┐   │
 │   │  Dedup & Lock   │ ────► │ Dispatcher / Workers │   │
 │   └─────────────────┘       └──────────────────────┘   │
 │            │                            │              │
 │            ▼                            ▼              │
 │   ┌─────────────────┐       [Integration Adapters]     │
 │   │ PostgreSQL (DB) │       - Habitica API             │
 │   └─────────────────┘       - Google Tasks API         │
 └─────────────────────────────- Google Calendar API──────┘
         ▲
         │  (HTTP Webhook via Google Apps Script)
 [Google Tasks / Calendar]
```

### Anti-Looping & Concurrency Mechanism

1. Sync Mutex / Origin Tracking: Every incoming mutation carries an `origin` header (e.g., `origin: "taskwarrior"`).
2. Deterministic Locking: The Hub hashes payload contents (`content_hash`). If an incoming payload matches the database's `content_hash` within a 5-second window, it is acknowledged and discarded.
3. Targeted Dispatch: Outbound sync dispatches only to adapters `where adapter.name !== payload.origin`.

## Database Schema (Prisma)

```typescript
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum TaskStatus {
  PENDING
  COMPLETED
  DELETED
}

enum SyncOrigin {
  TASKWARRIOR
  GOOGLE_TASKS
  HABITICA
  GOOGLE_CALENDAR
}

model TaskMapping {
  id              String      @id @default(uuid())
  title           String
  description     String?
  status          TaskStatus  @default(PENDING)
  dueDate         DateTime?
  lastModifiedIn  SyncOrigin
  contentHash     String      // MD5/SHA256 of canonical task fields for deduplication
  version         Int         @default(1)
  
  // External Identifiers
  taskwarriorUuid String?     @unique
  googleTaskId    String?     @unique
  habiticaId      String?     @unique
  googleEventId   String?     @unique
  
  metadata        Json?       // Platform-specific non-standard fields (e.g., tags, difficulty)
  
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  history         SyncLog[]

  @@index([status])
  @@index([lastModifiedIn])
}

model SyncLog {
  id            String      @id @default(uuid())
  taskMappingId String
  origin        SyncOrigin
  operation     String      // CREATE, UPDATE, DELETE
  rawPayload    Json
  success       Boolean     @default(true)
  errorMessage  String?
  timestamp     DateTime    @default(now())

  taskMapping   TaskMapping @relation(fields: [taskMappingId], references: [id], onDelete: Cascade)

  @@index([taskMappingId])
  @@index([timestamp])
}
```

## Integration Interfaces & Adapters

### Taskwarrior Hook Scripts (`on-add` and `on-modify`)

Iris is triggered by both task **creation** and task **modification** in Taskwarrior.

* Location: `~/.task/hooks/on-add.iris` and `~/.task/hooks/on-modify.iris` (both symlinks to the same script, which picks its mode from the file name).
* `on-add` Behavior: Reads the new task as one JSON line on standard input, echoes it back unchanged on standard output (required by Taskwarrior), and dispatches it to Iris as a `CREATE` event with the authentication bearer token and origin flag.
* `on-modify` Behavior: Reads two JSON lines (original task, modified task) on standard input, echoes the modified task back unchanged, checks for changes, and dispatches the modified task to Iris as an `UPDATE` event. Completion (`task done`) and deletion (`task delete`) arrive through `on-modify` as status changes.
* Failure Mode: Non-blocking in both hooks. The event is first written to a local spool file and then delivered by a detached background flush. If Iris is offline, the spool is retried later. The hook always exits `0`, so CLI execution never freezes and the task is never rejected.

### Google Tasks / Calendar Webhook

* Strategy: Google Apps Script configured with a Time-driven trigger (or Google Cloud Pub/Sub push notification for Google Calendar v3).
* Behavior: Scans modified tasks since `last_sync_timestamp` and pushes entries to `POST /webhooks/google-tasks`.

### Habitica REST Adapter

* Authentication: `x-api-user` and `x-api-key` headers.
* Task Mapping: Maps Iris `PENDING` -> `TODO`, and `COMPLETED` -> `POST /api/v3/tasks/{id}/score/up`.

## Implementation Roadmap

Phase 1: Core Hub & Data Contract

1. Initialize NestJS repository with `@prisma/client` and PostgreSQL container.
2. Run Prisma migration for `TaskMapping` and `SyncLog`.
3. Implement `SyncOrchestratorService` with hash-deduplication logic and transaction locking.

Phase 2: Taskwarrior & Local Sync

1. Develop the Taskwarrior `on-add` and `on-modify` Python hooks.
2. Build `TaskwarriorAdapter` in NestJS to parse UUIDs, tags, and status transformations.
3. Test personal PC and work PC cross-sync using the central mapping database.

Phase 3: External Integrations

1. Build `HabiticaAdapter` (handle score increments on complete, task deletion).
2. Configure Google Apps Script webhook for Google Tasks and Calendar events.
