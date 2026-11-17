# 05 — Dispatch worker, maintenance & admin API

**Goal:** deliver outbox jobs to push platforms reliably and idempotently; prune data; expose the command-center API.
**ADRs:** 004, 007, 013. **Docs:** [DBDD: `sync_jobs`](../docs/iris-dbdd.md#sync_jobs), [DBDD raw SQL](../docs/iris-dbdd.md#raw-sql-in-migrations) item 3.

## 1. Push adapter contract (`src/sync/push-adapter.ts`)

```ts
export const PUSH_ADAPTERS = Symbol('PUSH_ADAPTERS');

export interface PushContext {
  task: CanonicalFields & { id: string };
  link: { externalId: string; extra: Record<string, unknown> } | null;   // null on create
  tz: string;
}

export interface PushAdapter {
  readonly platform: PushPlatform;
  /** Create the remote item. MUST be idempotent per task.id (alias / extendedProperty / lookup). */
  create(ctx: PushContext): Promise<{ externalId: string; extra?: Record<string, unknown> }>;
  /** Overwrite every supported field. Throw RemoteNotFoundError if the item is gone. */
  update(ctx: PushContext & { link: NonNullable<PushContext['link']> }): Promise<{ extra?: Record<string, unknown> }>;
  /** Delete the remote item. Treat 404/410 as success. */
  delete(ctx: PushContext & { link: NonNullable<PushContext['link']> }): Promise<void>;
}
```

Error classes (from `src/common/errors.ts`): `RetryableError` (network, 429, 5xx, 401/403), `PermanentError` (other 4xx), `RemoteNotFoundError` (404/410 on update).

Provide `FakePushAdapter` in `test/helpers/` that records calls and can be scripted to throw.

## 2. `SyncWorkerService` (`src/sync/sync-worker.service.ts`)

Lifecycle: start a loop in `onApplicationBootstrap` if `SYNC_WORKER_ENABLED`; stop in `onApplicationShutdown` (await the in-flight batch). Loop: `tick()` then `setTimeout(SYNC_WORKER_INTERVAL_MS)`; never overlap ticks.

`tick()`:
1. Claim jobs with [DBDD raw SQL](../docs/iris-dbdd.md#raw-sql-in-migrations) item 3 (`LIMIT SYNC_WORKER_BATCH`).
2. Process claimed jobs **sequentially** (rate-limit friendliness) via `runJob(job)`.

`runJob(job)` (public for tests):
```text
adapter = adapters[job.target]; if missing → mark DEAD lastError='no adapter' → return
task = load task (with links)
f = toFields(task); link = task.links.find(target)
desired = shouldExistRemotely(target, f)

if !desired:
    if link?.state == ACTIVE && link.externalId:
        adapter.delete(...)
        link ← state DETACHED, lastSyncedHash = hashProjection(target, deletionCandidate(target, f), tz), lastSyncedAt=now
        log OUTBOUND DELETE APPLIED
    → DONE
else if !link || link.state != ACTIVE || !link.externalId:
    -- Habitica rule (ADR-011): if target==HABITICA && link?.state==DETACHED && f.status==COMPLETED → DONE (no re-create)
    upsert link state=PENDING_CREATE, externalId=null   ← COMMIT this before the remote call
    {externalId, extra} = adapter.create(ctx(link=null))
    link ← ACTIVE, externalId, extra ⊕, lastSyncedHash = hashProjection(target, f, tz), lastSyncedAt=now
    log OUTBOUND CREATE APPLIED → DONE
else:
    h = hashProjection(target, f, tz)
    if h == link.lastSyncedHash → DONE (already in sync; no log)
    try adapter.update(...)
    catch RemoteNotFoundError:
        if target==HABITICA && f.status==COMPLETED → link DETACHED → DONE
        else → treat as create branch (PENDING_CREATE then create)
    link ← lastSyncedHash=h, lastSyncedAt=now, extra ⊕ → log OUTBOUND UPDATE APPLIED → DONE
```

- **Re-read before writing the link:** after the remote call, update the link in a short transaction. Set `lastSyncedHash` to the hash of the state **that was pushed** (`f`), not a reloaded state. If the task changed meanwhile, a new PENDING job already exists and converges it.
- **DONE:** `status=DONE, lockedAt=null`.
- **Failure:** `RetryableError`/unknown → `attempts+1`; if `attempts ≥ 10` → `DEAD`, else `PENDING` with `nextRunAt = now + min(5s·2^attempts, 1h)` and `lastError` (message ≤ 500 chars, no secrets). `PermanentError` → `DEAD` immediately. Always log `OUTBOUND … FAILED` with `errorMessage`.
- **Partial unique conflict:** when resetting a failed job to `PENDING`, a newer PENDING job for the same `(task, target)` may already exist (unique violation). In that case mark this job `DONE` (the newer job supersedes it).

## 3. `MaintenanceService` (`@nestjs/schedule`)

| Schedule | Action |
| --- | --- |
| every minute | `UPDATE sync_jobs SET status='PENDING', locked_at=NULL WHERE status='RUNNING' AND locked_at < now() - interval '5 minutes'` |
| daily 04:00 (`IRIS_TIMEZONE`) | delete `sync_logs` older than 90 days; delete `sync_jobs` `DONE` older than 7 days |

Also run the stale-RUNNING reset once at boot. Disabled when `SYNC_WORKER_ENABLED=false`.

## 4. Admin API (`src/admin/`, all `@RequirePrincipal('admin')`)

| Method & path | Behavior |
| --- | --- |
| `GET /v1/tasks?status=&q=&limit=50&cursor=` | Tasks ordered by `updatedAt desc, id`; `q` = case-insensitive `title` contains; keyset cursor = opaque base64url of `{updatedAt,id}`. Each item includes `links: [{platform, externalId, state, lastSyncedAt}]`. Returns `{ items, nextCursor }`. |
| `GET /v1/tasks/:id` | Task + links + last 50 `SyncLog` (desc) + non-DONE jobs. 404 if missing. |
| `POST /v1/tasks/:id/resync` | Clears `lastSyncedHash` on the task's push links and enqueues all enabled push targets. 202. |
| `GET /v1/sync/jobs?status=PENDING\|RUNNING\|DEAD&limit=50` | Jobs with `taskId`, `target`, `attempts`, `nextRunAt`, `lastError`. |
| `POST /v1/sync/jobs/:id/retry` | Only from `DEAD` → `PENDING`, `attempts=0`, `nextRunAt=now`. 409 if not DEAD or a PENDING job for the same `(task,target)` exists. 202 on success. |

Query params are validated with zod (`limit` 1–200).

## Tests

- Unit (`sync-worker.service.spec.ts`, Prisma mocked or e2e DB — your choice, but cover every branch):
  create; update; skip-when-hash-equal; delete when DELETED; RemoteNotFound → re-create; Habitica completed + NotFound → DETACHED without create; Habitica DETACHED + completed → no create; retry backoff math; DEAD after 10; PermanentError → DEAD; PENDING_CREATE committed before `create()` is called (assert via the fake adapter reading the DB inside `create`). Spec 08 (post-MVP) adds: calendar delete when due cleared.
- E2E `dispatch.e2e-spec.ts`: ingest TW change → run `tick()` with fakes for both MVP targets (Google Tasks, Habitica) → links ACTIVE with hashes; then ingest a GT echo of the pushed state → NOOP_ECHO.
- E2E `admin.e2e-spec.ts`: list/filter/paginate, detail, retry state machine (202/409), resync enqueues.

## Done when

All tests pass; manual run with `SYNC_WORKER_ENABLED=true` + fake adapters bound in dev shows jobs draining (log lines per job).
