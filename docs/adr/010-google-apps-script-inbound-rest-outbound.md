---
type: adr
project: iris
number: 010
status: accepted
updated: 2026-11-16
---

# 010. Google Tasks & Calendar: Apps Script inbound, REST API outbound

## Context

Google Tasks has no push notifications. Calendar push channels need a verified domain and renewal.

## Decision

- **Inbound:** one Apps Script project with a 1-minute time-driven trigger. For Tasks, it lists `updatedMin = lastRun − 60s` with `showDeleted/showHidden/showCompleted`. For Calendar, it uses `Events.list` with a `syncToken` on **one dedicated calendar** (`Iris`). Changes are POSTed in batches to `/v1/ingest/google-tasks` and `/v1/ingest/google-calendar`.
- **Outbound:** Iris calls the Tasks v1 and Calendar v3 REST APIs with an OAuth2 refresh token (scopes `tasks`, `calendar.events`). A one-time loopback CLI (`npm run google:auth`) obtains the token. Iris syncs **one configured task list** (`GOOGLE_TASKLIST_ID`) and **one calendar** (`GOOGLE_CALENDAR_ID`).
- **Identity:** calendar events carry `extendedProperties.private.irisTaskId`. Google Tasks cannot carry a client ID, so a link sits in `PENDING_CREATE` during insert, and an inbound item with an unknown ID whose projected hash matches a `PENDING_CREATE` link is adopted instead of creating a duplicate.
- **Calendar projection:** only non-deleted tasks with a due are events. An all-day due becomes an all-day event; a timed due becomes a `[due, due + GOOGLE_CALENDAR_EVENT_MINUTES]` event (default 30).

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

Inbound latency is about 1–2 minutes, and personal calendars are never imported.
