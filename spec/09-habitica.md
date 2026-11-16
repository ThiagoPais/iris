# 09 — Habitica (todos)

**Goal:** every task is a Habitica todo; completing it anywhere scores it up (XP/gold), reopening scores it down.
**ADRs:** 011, 012, 013. **Docs:** PRD §5 (Habitica column).
**Dependencies:** none (use global `fetch`).

## 1. Client (`src/platforms/habitica/habitica.client.ts`)

- Base URL `https://habitica.com/api/v3`. Headers on every call: `x-api-user`, `x-api-key`, `x-client: <HABITICA_USER_ID>-iris`, `content-type: application/json`.
- Throttle: token bucket of 25 requests per rolling 60 s, shared by all calls; callers await a slot.
- Response envelope `{ success, data, message? }`. Return `data`.
- Errors: network / 429 / 5xx / 401 → `RetryableError`; 404 → `RemoteNotFoundError`; other 4xx → `PermanentError` (message = Habitica `message`, never headers).
- Interface for faking:
```ts
export interface HabiticaApi {
  createTodo(body: HabiticaTodoBody): Promise<HabiticaTask>;           // POST /tasks/user
  getTask(idOrAlias: string): Promise<HabiticaTask>;                   // GET  /tasks/{idOrAlias}
  updateTask(id: string, body: Partial<HabiticaTodoBody>): Promise<HabiticaTask>; // PUT /tasks/{id}
  score(id: string, dir: 'up' | 'down'): Promise<void>;                // POST /tasks/{id}/score/{dir}
  deleteTask(id: string): Promise<void>;                               // DELETE /tasks/{id}
  listWebhooks(): Promise<HabiticaWebhook[]>;                          // GET  /user/webhook
  createWebhook(b: object): Promise<void>;                             // POST /user/webhook
  updateWebhook(id: string, b: object): Promise<void>;                 // PUT  /user/webhook/{id}
}
type HabiticaTodoBody = { type: 'todo'; text: string; notes: string; priority: 0.1|1|1.5|2; date: string | null; alias?: string };
```

## 2. Adapter (`habitica.adapter.ts`, `platform = 'HABITICA'`)

Render: `text = title`, `notes = notes ?? ''`, `priority = {NONE:0.1, LOW:1, MEDIUM:1.5, HIGH:2}`, `date = due ? dueDate(due, tz) : null`, `alias = 'iris-' + task.id`.

- **create**: `createTodo(body)`. On `PermanentError` whose message mentions the alias (duplicate) → `getTask('iris-' + id)` and continue with that task (crash idempotency). Then, if `status === COMPLETED`, `score(id,'up')`. Return `{ externalId: t.id, extra: { completed: status === COMPLETED } }`.
- **update**: `updateTask(id, {text, notes, priority, date})`. Then reconcile completion against `link.extra.completed`: COMPLETED and not completed → `score up`; PENDING and completed → `score down`. Return `{ extra: { completed } }`.
- **delete**: `deleteTask`; swallow 404.
- Clearing a date: send `date: null`. If Habitica rejects it (400), retry once with `date: ''` and record the finding in `spec/QUESTIONS.md`.
- The worker handles the "no re-create for completed + DETACHED/NotFound" rules (spec 05).

## 3. Inbound (`habitica.mapper.ts` + controller)

Route: `POST /v1/ingest/habitica/:secret`, `@Public()`. `safeEqual(secret, env.HABITICA_WEBHOOK_SECRET)` false → **404**. **Always respond 200** once authenticated, even for ignored items (Habitica disables failing webhooks).

Payload (`taskActivity`): `{ type: 'created'|'updated'|'deleted'|'scored', direction?: 'up'|'down', task: { id, type, text, notes, priority, date, completed, alias?, updatedAt }, user: { _id } }`.
- `user._id !== env.HABITICA_USER_ID` → ignore.
- `task.type !== 'todo'` → ignore.
- `irisTaskId` = `alias` matching `^iris-([0-9a-f-]{36})$`.
- `deleted` → `op: DELETE`.
- Otherwise `op: UPSERT` with fields `title = text`, `notes = notes || null`, `priority` reverse map (nearest of 0.1/1/1.5/2), `due = date ? allDayDue(toLocalDate(new Date(date), tz)) : null`, and `status`: for `scored`, `direction === 'up' ? COMPLETED : PENDING`; otherwise `completed ? COMPLETED : PENDING`.
- `sourceUpdatedAt = updatedAt ? new Date(updatedAt) : new Date()`. `linkExtra = { completed: status === COMPLETED }`.
- `rawPayload` = the payload with `user` reduced to `{ _id }` (the user object contains stats and profile data).

## 4. Webhook registration (`scripts/habitica-webhook.ts`)

`node scripts/habitica-webhook.ts <public base url>`: builds `url = <base>/v1/ingest/habitica/<HABITICA_WEBHOOK_SECRET>`, then lists webhooks. If one with `label === 'iris'` exists → update it; else create `{ url, label: 'iris', type: 'taskActivity', enabled: true, options: { created: true, updated: true, deleted: true, scored: true } }`. Prints the URL **with the secret masked**. npm script: `"habitica:webhook"`.

## Tests

- Client: the throttle allows 25 calls then waits (fake timers); envelope unwrap; error classification.
- Adapter with a fake `HabiticaApi`: create + completed → score up; update reconcile (up/down/none); alias duplicate → `getTask` fallback; priority mapping both ways.
- Mapper: `scored` up/down; `deleted`; non-todo ignored; wrong user ignored; alias hint parsed.
- E2E: wrong secret → 404; `scored up` webhook → task COMPLETED, GT/CAL jobs enqueued; echo of Iris's own score → NOOP_ECHO; Iris-created todo webhook arriving **before** the worker commits the link → resolved by alias (no duplicate task).

## Done when

Tests pass; manual: US-02 (tick in Google Tasks → Habitica rewards).
