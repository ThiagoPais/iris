# 08 — Google Calendar (dedicated `Iris` calendar)

**Goal:** tasks with a due date appear as events on one calendar. Moving an event reschedules its task; deleting it unschedules the task.
**ADRs:** 010, 013, 014. **Depends on:** spec 07 (auth, error mapping, Apps Script base).
**Dependency:** `@googleapis/calendar`.

## 1. Adapter (`src/platforms/google/calendar/`)

```ts
export interface GoogleCalendarApi {
  findByIrisId(calendarId: string, taskId: string): Promise<GEvent | null>;  // events.list privateExtendedProperty=irisTaskId=<id>, showDeleted=false, maxResults=1
  insert(calendarId: string, body: GEventBody): Promise<GEvent>;
  patch(calendarId: string, id: string, body: GEventBody): Promise<GEvent>;
  delete(calendarId: string, id: string): Promise<void>;
}
```

Render `GEventBody` from `ctx.task`:
- `summary = title`, `description = notes ?? ''`.
- All-day: `start = { date: d }`, `end = { date: d + 1 day }` where `d = dueDate(due, tz)`.
- Timed: `start = { dateTime: due.at.toISOString(), timeZone: tz }`, `end = start + minutes`, where `minutes = link?.extra.durationMinutes ?? env.GOOGLE_CALENDAR_EVENT_MINUTES`.
- `extendedProperties = { private: { irisTaskId: task.id } }`.
- Never set attendees, reminders or colors (the user's edits to those survive, because `patch` leaves them untouched).

Operations:
- `create`: first `findByIrisId`. If found → `patch` it and return its id (idempotency after a crash); else `insert`. Return `{ externalId: ev.id, extra: { etag: ev.etag } }`.
- `update` → `patch`. `delete` → `delete`; swallow 404/410.
- Use `classifyGoogleError` from spec 07.

The worker never calls the calendar adapter for a task without a due (`shouldExistRemotely` handles it: an existing event is deleted when the due is cleared).

## 2. Inbound mapper (`google-calendar.mapper.ts`, pure)

Input: a raw Calendar v3 event.
- `recurringEventId` present or `recurrence` present → `ignore('recurring event')`.
- `status === 'cancelled'` → `op: DELETE`, `fields: {}` (ingest turns it into `due = null`, ADR-013).
- Else `op: UPSERT` with fields `title = summary ?? ''` (empty → `ignore('empty title')`), `notes = description || null`, and `due`:
  - `start.date` → `allDayDue(start.date)`.
  - `start.dateTime` → `{ at: new Date(start.dateTime), allDay: false }`.
- `linkExtra = { etag, durationMinutes }`, where `durationMinutes = (end − start)/60000` for timed events, omitted for all-day.
- `irisTaskId = extendedProperties?.private?.irisTaskId`. `sourceUpdatedAt = new Date(updated)`.

## 3. Endpoint (`@RequirePrincipal('google-script')`)

`POST /v1/ingest/google-calendar`, body `{ calendarId: string, items: GEvent[] (max 500) }`. `calendarId !== env.GOOGLE_CALENDAR_ID` → 400. Map, `ingestMany`, `200 { results }`.

## 4. Apps Script addition (`Code.gs`)

Script Properties: `CALENDAR_ID`, `CAL_SYNC_TOKEN`.

```js
function syncCalendar_() {
  const p = PropertiesService.getScriptProperties();
  const calId = p.getProperty('CALENDAR_ID');
  let token = p.getProperty('CAL_SYNC_TOKEN');
  let pageToken, items = [], res;
  const base = { showDeleted: true, maxResults: 250 };
  try {
    do {
      const opts = token ? Object.assign({ syncToken: token, pageToken }, base)
                         : Object.assign({ timeMin: new Date(Date.now() - 30 * 86400000).toISOString(), pageToken }, base);
      res = Calendar.Events.list(calId, opts);
      items = items.concat(res.items || []);
      pageToken = res.nextPageToken;
    } while (pageToken);
  } catch (e) {
    if (String(e).indexOf('410') >= 0) { p.deleteProperty('CAL_SYNC_TOKEN'); return syncCalendar_(); }
    throw e;
  }
  for (let i = 0; i < items.length; i += 100)
    post_('/v1/ingest/google-calendar', { calendarId: calId, items: items.slice(i, i + 100) });
  if (res.nextSyncToken) p.setProperty('CAL_SYNC_TOKEN', res.nextSyncToken);
}
```

(`singleEvents` is **not** set, so recurring series arrive as masters and are ignored by the mapper.)

## Tests

- Mapper: all-day, timed (duration computed), cancelled → DELETE, recurring ignored, `irisTaskId` hint extracted.
- Adapter with a fake API: all-day `end` is the next day (crossing a month boundary); timed uses the stored duration, otherwise the default; `create` patches when `findByIrisId` hits.
- E2E:
  - A TW task with a due → worker inserts an event.
  - A calendar event moved to Friday 14:00 → task due is timed Friday 14:00; GT and Habitica jobs are enqueued.
  - A cancelled event → task due null, task still PENDING, CAL link DETACHED.
  - A due set again later → a new event is created.

## Done when

Tests pass; manual: US-05 and US-06 from the PRD behave as described.
