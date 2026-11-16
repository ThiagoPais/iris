# 07 — Google OAuth, Google Tasks & Apps Script (Tasks part)

**Goal:** bidirectional sync with one Google Tasks list: Apps Script pushes inbound, Iris calls the REST API outbound.
**ADRs:** 004, 006, 010, 014. **Docs:** PRD §5 (Google Tasks column).
**Dependencies:** `@googleapis/tasks`, `google-auth-library`.

## 1. OAuth bootstrap (`scripts/google-auth.ts`)

Run with `node scripts/google-auth.ts` (Node ≥ 24 type stripping; no `.js` imports needed in this standalone file). It reads `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` from the env (Desktop OAuth client), starts a loopback server on `127.0.0.1:<random port>`, prints the consent URL (`access_type=offline`, `prompt=consent`, scopes `https://www.googleapis.com/auth/tasks` and `https://www.googleapis.com/auth/calendar.events`), exchanges the code, prints `GOOGLE_REFRESH_TOKEN=<token>` and exits. Add the npm script `"google:auth": "node scripts/google-auth.ts"`.

## 2. `GoogleAuthService` (`src/platforms/google/google-auth.service.ts`)

Singleton `OAuth2Client` with the refresh token. Exposes `client()`. It is only constructed when a `GOOGLE_*` push target is enabled (conditional provider).

## 3. Adapter (`src/platforms/google/tasks/`)

Wrap the API behind an interface so tests can fake it:
```ts
export interface GoogleTasksApi {
  insert(listId: string, body: GTaskBody): Promise<GTask>;
  patch(listId: string, id: string, body: GTaskBody): Promise<GTask>;
  delete(listId: string, id: string): Promise<void>;
}
type GTaskBody = { title: string; notes: string | null; status: 'needsAction'|'completed'; due: string | null; completed?: null };
```

`GoogleTasksAdapter implements PushAdapter` (`platform = 'GOOGLE_TASKS'`):
- Render: `title`; `notes`; `status = COMPLETED ? 'completed' : 'needsAction'` (with `completed: null` when `needsAction`); `due = due ? dueDate(due, tz) + 'T00:00:00.000Z' : null`.
- `create` → `insert` → `{ externalId: res.id }`. `update` → `patch` (send every field; `null` clears). `delete` → `delete`; swallow 404/410.
- Error mapping (`classifyGoogleError`, shared with spec 08): no response / 429 / 5xx / 401 / 403 → `RetryableError`; 404 / 410 on patch → `RemoteNotFoundError`; other 4xx → `PermanentError`. Messages must not include request headers.

## 4. Inbound mapper (`google-tasks.mapper.ts`, pure)

Input: a raw Tasks API resource `{ id, title?, notes?, status, due?, completed?, deleted?, hidden?, updated }`.
- `deleted === true` → `op: DELETE`, `fields: {}`.
- Else if `(title ?? '').trim() === ''` → `ignore('empty title')` (Google creates blank rows while typing).
- Else `op: UPSERT`, fields: `title`, `notes: notes ?? null`, `status: status === 'completed' ? 'COMPLETED' : 'PENDING'`, `due: due ? allDayDue(due.slice(0, 10)) : null`.
- `sourceUpdatedAt = new Date(updated)`; `completedAt = completed ? new Date(completed) : null`; `externalId = id`.
- `hidden` is ignored (cleared completed tasks keep their state).

## 5. Endpoint (`@RequirePrincipal('google-script')`)

`POST /v1/ingest/google-tasks`, body `{ taskListId: string, items: GTask[] (max 500) }`. `taskListId !== env.GOOGLE_TASKLIST_ID` → 400 `wrong task list`. Then map and `ingestMany` → `200 { results: [{ id, outcome, taskId }] }`.

## 6. Apps Script (`integrations/google-apps-script/`)

`appsscript.json`: `timeZone` = same as `IRIS_TIMEZONE`, `runtimeVersion: "V8"`, advanced services `Tasks v1` and `Calendar v3` (the Calendar part is used in spec 08), `oauthScopes`: tasks, calendar.readonly, script.external_request, script.scriptapp.

`Code.gs` (Tasks part; spec 08 adds `syncCalendar_`). **Script Properties:** `IRIS_URL`, `IRIS_TOKEN`, `TASKLIST_ID`, `TASKS_LAST_RUN`.

```js
function syncAll() {                       // time-driven trigger target
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(5000)) return;
  try { syncTasks_(); if (typeof syncCalendar_ === 'function') syncCalendar_(); }
  finally { lock.releaseLock(); }
}
function syncTasks_() {
  const p = PropertiesService.getScriptProperties();
  const startedAt = new Date();
  const last = p.getProperty('TASKS_LAST_RUN');
  const updatedMin = new Date((last ? Date.parse(last) : Date.now() - 86400000) - 60000).toISOString();
  let pageToken, batch = [];
  do {
    const res = Tasks.Tasks.list(p.getProperty('TASKLIST_ID'),
      { updatedMin, showDeleted: true, showHidden: true, showCompleted: true, maxResults: 100, pageToken });
    batch = batch.concat(res.items || []);
    pageToken = res.nextPageToken;
  } while (pageToken);
  for (let i = 0; i < batch.length; i += 100)
    post_('/v1/ingest/google-tasks', { taskListId: p.getProperty('TASKLIST_ID'), items: batch.slice(i, i + 100) });
  p.setProperty('TASKS_LAST_RUN', startedAt.toISOString());   // only reached if every post succeeded
}
function post_(path, body) {
  const p = PropertiesService.getScriptProperties();
  const r = UrlFetchApp.fetch(p.getProperty('IRIS_URL') + path, {
    method: 'post', contentType: 'application/json', muteHttpExceptions: true,
    headers: { Authorization: 'Bearer ' + p.getProperty('IRIS_TOKEN') }, payload: JSON.stringify(body) });
  if (r.getResponseCode() >= 300) throw new Error('Iris ' + path + ' -> ' + r.getResponseCode());
}
function install() {                        // run once manually
  ScriptApp.getProjectTriggers().forEach(t => ScriptApp.deleteTrigger(t));
  ScriptApp.newTrigger('syncAll').timeBased().everyMinutes(1).create();
}
function backfillTasks() {                  // run manually: resend everything
  PropertiesService.getScriptProperties().setProperty('TASKS_LAST_RUN', '1970-01-02T00:00:00Z');
  syncTasks_();
}
```

`README.md` in that folder: create the project at script.google.com (or with `clasp`), paste the files, set the Script Properties, enable the advanced services, run `install()` once, and authorize.

## Tests

- Mapper unit: deleted; blank title ignored; completed; due date-only; notes null.
- Adapter unit with a fake `GoogleTasksApi`: render payloads (timed due → local date; `needsAction` clears `completed`); error classification table.
- E2E: POST inbound → task created; worker `tick()` with the adapter + fake API → `patch` payload matches; echo POST → NOOP_ECHO; wrong list → 400; agent token → 403.
- Apps Script: no automated tests; `README.md` contains a manual checklist (create, complete, delete in Android → Iris logs APPLIED).

## Done when

Tests pass; manual: a task added in Google Tasks on Android appears in Taskwarrior after one Apps Script run and one agent sync.
