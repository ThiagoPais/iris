# 04 — Ingest, outbox enqueue & change feed

**Goal:** a single transactional entry point that turns any platform's change into canonical state. It suppresses echoes and duplicates, rejects stale edits, records audit and feed rows, and enqueues outbound jobs.
**ADRs:** 003, 004, 005, 006, 007, 008, 013. **Docs:** DDD §4–§5.

## 1. Contracts (`src/sync/types.ts`)

```ts
export interface InboundChange {
  platform: Platform;                     // set by the controller from the route (I3)
  externalId: string;                     // platform-native id (TW uuid, Google id, Habitica id…)
  op: 'UPSERT' | 'DELETE';
  fields: Partial<CanonicalFields>;       // normalized; full-payload platforms set EVERY supported key (null to clear)
  sourceUpdatedAt: Date;                  // platform's modification time
  completedAt?: Date | null;              // platform's completion time, if known
  irisTaskId?: string;                    // identity hint (Habitica alias, Calendar extendedProperty)
  deviceId?: string;                      // agent platforms only
  linkExtra?: Record<string, unknown>;    // shallow-merged into ExternalLink.extra when applied/noop
  rawPayload: unknown;                    // stored in SyncLog (secrets already removed)
}

export type IngestOutcome = 'APPLIED' | 'NOOP_ECHO' | 'NOOP_DUPLICATE' | 'REJECTED_STALE' | 'REJECTED_INVALID' | 'IGNORED';
export interface IngestResult { outcome: IngestOutcome; taskId: string | null; version: number | null; reason?: string }

// Implemented per platform (specs 06–10). Pure: no DB, no I/O.
export interface InboundMapper<P> {
  toInbound(payload: P, ctx: { tz: string; deviceId?: string }):
    | { kind: 'change'; change: Omit<InboundChange, 'platform'> }
    | { kind: 'ignore'; reason: string };
}
```

## 2. `OutboxService` (`src/sync/outbox.service.ts`)

`enqueue(tx, taskId: string, targets: PushPlatform[]): Promise<void>`. For each target in `targets ∩ env.IRIS_PUSH_TARGETS`, run raw SQL:

```sql
INSERT INTO sync_jobs (id, task_id, target, status, attempts, next_run_at, created_at, updated_at)
VALUES ($1, $2, $3::"Platform", 'PENDING', 0, now(), now(), now())
ON CONFLICT (task_id, target) WHERE status = 'PENDING'
DO UPDATE SET next_run_at = LEAST(sync_jobs.next_run_at, now()), updated_at = now();
```

`tx` is the Prisma interactive-transaction client.

## 3. `IngestService.ingest(change: InboundChange): Promise<IngestResult>`

Wrap everything in `prisma.$transaction(async tx => …, { timeout: 10_000 })`. `tz = env.IRIS_TIMEZONE`. `ECHOING = PUSH_PLATFORMS`.

```text
S0  if change.fields has keys: fields = normalizeFields(change.fields)
    catch DomainValidationError → log REJECTED_INVALID (taskId null) → return
S1  SELECT pg_advisory_xact_lock(hashtextextended($platform || ':' || $externalId, 0))
S2  link = ExternalLink where (platform, externalId)
S3  if !link && change.irisTaskId && Task(irisTaskId) exists:
        link = upsert ExternalLink(taskId=irisTaskId, platform) set externalId, state=ACTIVE
S4  if !link && platform ∈ ECHOING:                       -- PENDING_CREATE adoption (ADR-010)
        for L in ExternalLink where platform, state=PENDING_CREATE, createdAt > now()-15min:
          t = load L.task
          if hashProjection(platform, mergeInbound(t, fields, platform, tz), tz) == hashProjection(platform, t, tz):
            link = update L set externalId, state=ACTIVE; break
S5  if !link:
        if op == DELETE → log IGNORED('unknown item delete') → return
        f = mergeInbound(null, fields, platform, tz)       -- throws → REJECTED_INVALID
        create Task(f, contentHash, version=1, lastEditAt=sourceUpdatedAt,
                    lastEditOrigin=platform, lastEditDevice=deviceId,
                    completedAt = f.status==COMPLETED ? (change.completedAt ?? sourceUpdatedAt) : null)
        create ExternalLink(platform, externalId, ACTIVE,
                    lastSyncedHash = platform∈ECHOING ? hashProjection(platform,f,tz) : null,
                    remoteUpdatedAt=sourceUpdatedAt, extra=linkExtra)
        appendChange(tx, task, platform, deviceId)          -- §4
        log APPLIED/CREATE; enqueue(tx, task.id, PUSH_PLATFORMS \ {platform}); return
S6  task = SELECT * FROM tasks WHERE id=link.taskId FOR UPDATE
    cur = toFields(task)
    cand = op==DELETE ? deletionCandidate(platform, cur) : mergeInbound(cur, fields, platform, tz)
S7  ECHO: if platform∈ECHOING && link.lastSyncedHash != null
            && hashProjection(platform, cand, tz) == link.lastSyncedHash:
        update link remoteUpdatedAt, extra ⊕ linkExtra → log NOOP_ECHO → return
S8  STALE: if sourceUpdatedAt < task.lastEditAt − 2000ms:
        log REJECTED_STALE; if platform∈ECHOING: enqueue(tx, task.id, [platform]) → return
S9  DUPLICATE: if contentHash(cand, tz) == task.contentHash:
        update link: lastSyncedHash = platform∈ECHOING ? hashProjection(platform,cand,tz) : null,
                     remoteUpdatedAt, extra ⊕ linkExtra
        log NOOP_DUPLICATE → return
S10 APPLY:
        completedAt = cand.status==COMPLETED
            ? (cur.status==COMPLETED ? task.completedAt : (change.completedAt ?? sourceUpdatedAt))
            : null
        update Task(cand, completedAt, contentHash, version+1,
                    lastEditAt = max(task.lastEditAt, sourceUpdatedAt),
                    lastEditOrigin=platform, lastEditDevice=deviceId ?? null)
        update link: lastSyncedHash (as S9), remoteUpdatedAt, extra ⊕ linkExtra,
                     state = (op==DELETE && platform∈ECHOING) ? DETACHED : ACTIVE
        appendChange(tx, task, platform, deviceId)
        log APPLIED (operation: DELETE if op==DELETE or cand.status==DELETED else UPDATE)
        enqueue(tx, task.id, PUSH_PLATFORMS \ {platform})
```

Notes:
- `log` = insert `SyncLog { direction: INBOUND, platform, operation, outcome, rawPayload, taskId }`. For a new task, operation is `CREATE`.
- A `DETACHED` link found in S2 still goes through S6–S10. An `UPSERT` on a DETACHED link sets it `ACTIVE`.
- Prisma unique violation (P2002) on link creation (race across different externalIds) → retry the whole transaction once.
- Any other exception → rethrow (the controller maps it to 500; the agent or script retries).

## 4. Change feed (`src/sync/change-feed.service.ts`)

`appendChange(tx, task, origin, deviceId?)`:
```sql
SELECT pg_advisory_xact_lock(7461001);   -- global: ids are committed in id order (feed safety)
INSERT INTO task_changes (task_id, version, origin, origin_device) VALUES (...);
```
Take this lock **last** in the transaction (immediately before commit) to keep contention short.

`changesSince(platform: AgentPlatform, deviceId: string, since: bigint, limit = 200)`:
1. In one transaction:
   ```sql
   SELECT task_id, MAX(id) AS last_id FROM task_changes
   WHERE id > $since AND origin_device IS DISTINCT FROM $deviceId
   GROUP BY task_id ORDER BY last_id LIMIT $limit
   ```
2. `cursor` = if rows.length == limit → max(`last_id`) of the rows; else `(SELECT COALESCE(MAX(id), $since) FROM task_changes)`.
3. Load those tasks with their `platform` link. A task without a link gets one created (`ACTIVE`, `externalId = randomUUID()`, `lastSyncedHash = null`) (ADR-008).
4. Upsert `AgentDevice { id: deviceId, platform, cursor, lastSeenAt: now() }`.
5. Return `{ cursor: string, tasks: Array<{ task: TaskRecord; link: ExternalLinkRecord }> }`. Rendering to the platform's format happens in the controller (specs 06, 10).

## 5. Batch helper

`IngestService.ingestMany(changes)`: processes sequentially and returns an `IngestResult[]` in the same order. An exception in one item yields `{ outcome: 'REJECTED_INVALID', reason: 'internal error' }` for that item, logs it with `Logger.error`, and continues.

## Tests (e2e against Postgres, `test/ingest.e2e-spec.ts`; `IRIS_PUSH_TARGETS=GOOGLE_TASKS,HABITICA,GOOGLE_CALENDAR`)

1. New TASKWARRIOR upsert → task v1, link ACTIVE, 1 TaskChange, SyncLog APPLIED/CREATE, 3 PENDING jobs.
2. Same payload again → NOOP_DUPLICATE, still 3 jobs (coalesced), version 1.
3. Echo: set the GOOGLE_TASKS link `lastSyncedHash = hashProjection(GT, task)`, ingest a GT change with the same projection and a newer `sourceUpdatedAt` → NOOP_ECHO, task unchanged.
4. Stale: GT change older than `lastEditAt − 2s` → REJECTED_STALE and a GT job enqueued.
5. Partial merge: a GT change of the title only keeps tags and priority set by TW.
6. Calendar DELETE → due cleared, status unchanged, CAL link DETACHED.
7. Habitica DELETE → status DELETED, link DETACHED, jobs for GT and CAL.
8. `irisTaskId` hint attaches the link to an existing task (no new task).
9. PENDING_CREATE adoption: a GT link with no id + an unknown-id GT change with an identical projection → link adopted, no new task.
10. Unknown-id DELETE → IGNORED.
11. Empty title → REJECTED_INVALID, no task.
12. Feed: two devices. Changes from device A are not returned to A but are returned to B; the cursor advances past A's own changes; a missing TW link is minted once and reused.
13. Concurrency: 10 parallel identical new upserts (same externalId) → exactly 1 task.

## Done when

All the above pass. `IngestService` and `ChangeFeedService` are exported from a `SyncModule`.
