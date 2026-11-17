---
type: prd
project: iris
status: active
updated: 2026-11-17
---

# Iris: Product Requirements

## Overview

**Project Name:** Iris
**Description:** A personal sync hub and command center. It keeps one canonical list of tasks consistent across Taskwarrior (personal and work PCs), Google Tasks (Android), Google Calendar (through Google Tasks) and Habitica (gamification). It keeps cross-platform ID mappings, resolves edit collisions and never loops.

**Owner / Users:** One user (ADR-016).

### Problem

Tasks are captured wherever the user happens to be: the CLI at a desk, the phone on the go, the calendar when planning. Each tool holds a partial, drifting copy. Completing a task in one place does not complete it elsewhere, and Habitica rewards are only earned when the task is re-entered by hand.

## Goals

| # | Goal | Success metric |
| --- | --- | --- |
| G1 | A task created on any platform appears on all others | ≥ 99% of creates propagated; p95 latency ≤ 2 min (push targets), ≤ 1 poll interval (agents) |
| G2 | Completion, edits and deletions propagate | Same as G1 |
| G3 | No duplicates and no sync loops | 0 duplicate tasks per 1,000 syncs; 0 unbounded echo chains |
| G4 | No lost edits when a platform or Iris is offline | Every hook event eventually ingested; every push retried until success or `DEAD` |
| G5 | Zero friction on the CLI | Taskwarrior hooks add ≤ 100 ms whether Iris is up or down (no network I/O in the hook); they never fail a command |

## Users

| Persona | Needs | Notes |
|---|---|---|
| Owner | One consistent task list across two PCs (Taskwarrior), an Android phone (Google Tasks), Google Calendar (where dated Google Tasks show up) and Habitica, with no re-entry by hand. Sees what failed to sync and fixes it. | The only user (ADR-016). Also operates the command center. |

## Functional requirements

| ID | Requirement | Priority | Phase |
|---|---|---|---|
| FR-01 | A task created in Taskwarrior on either PC is sent to Iris. | Must | 2 |
| FR-02 | Edits, completion, deletion, start/stop and annotations made in Taskwarrior are sent to Iris. Only changes to fields Iris supports have an effect. | Must | 2 |
| FR-03 | Taskwarrior commands never wait on the network and never fail because of Iris. Changes made while Iris is unreachable are delivered when it returns. | Must | 2 |
| FR-04 | Each PC receives changes made elsewhere within one poll interval (60 s) and applies them without sending them back. A task keeps the same identifier on every PC. | Must | 2 |
| FR-05 | A one-time bootstrap sends every existing pending Taskwarrior task to Iris. | Should | 2 |
| FR-06 | Tasks tagged `+nosync` in Taskwarrior are never sent to other platforms. | Must | 2 |
| FR-07 | Tasks sync both ways with one configured Google Tasks list. Changes made on the phone reach Iris within 2 minutes. | Must | 3 |
| FR-08 | (removed) Tasks are not synced to Google Calendar as events. Google Calendar already shows Google Tasks items with a due date, so FR-07 covers it. Event sync is FR-15. | — | — |
| FR-09 | (removed) Depended on calendar events (FR-08). Rescheduling a task from Google Calendar changes its Google Tasks date and is covered by FR-07. | — | — |
| FR-10 | Tasks appear as Habitica todos. Completing a task anywhere scores it up (XP and gold), reopening scores it down, and priority maps to difficulty. | Must | 3 |
| FR-11 | The owner can list and search tasks and see each task's sync state on every platform. | Must | 1 |
| FR-12 | The owner can view a task's sync history. | Must | 1 |
| FR-13 | The owner can list outbound syncs that failed permanently and retry them. | Must | 1 |
| FR-14 | A health check reports whether storage is reachable and how many outbound syncs are waiting. | Must | 1 |
| FR-15 | A task marked as an event (for example tagged `+event`) syncs to Google Calendar as an event instead of a Google Tasks item. Details are open (Q4). | Could | Post-MVP |

## Non-functional requirements

| ID | Requirement | Measure |
|---|---|---|
| NFR-01 | Reliability: every change is delivered at least once, inbound and outbound, and a repeated delivery never creates a duplicate. | 0 duplicate tasks per 1,000 syncs (G3) |
| NFR-02 | Performance: Iris processes an incoming change quickly. | p95 ≤ 100 ms, measured at the server |
| NFR-03 | CLI friction: Taskwarrior hooks add little time to a command, whether Iris is up or down. | ≤ 100 ms per command (G5) |
| NFR-04 | Security: all traffic is encrypted, each source has its own credential, and no secret appears in any log, stored payload or error message. | 0 secrets found in logs and stored payloads |
| NFR-05 | Observability: every sync outcome can be traced per task and platform. | Sync history kept for 90 days |
| NFR-06 | Iris stays within each platform's API limits. | Habitica ≤ 25 requests/min; backs off when Google throttles |

## Out of scope

- Multiple users, sharing or collaboration.
- Habitica habits, dailies and rewards.
- Syncing tasks to Google Calendar as events in the MVP (planned later as FR-15).
- Recurring-task semantics: recurrence templates are not synced, their generated instances are.
- Field-level merge of concurrent edits (ADR-005).
- A graphical UI. The command center is a JSON admin API; a UI is a future project.
- Per-platform filters beyond Taskwarrior's `+nosync` tag.

## Success criteria

- US-01 to US-10 pass manually against the real Taskwarrior, Google and Habitica accounts.
- After 4 weeks of daily use, no task had to be re-entered by hand and no duplicate task appeared.

## Constraints and assumptions

- The PCs need only Python 3.11+ and Taskwarrior 2.6+/3.x.
- Google and Habitica must be able to reach Iris over the public internet (ADR-015).
- Google Calendar shows Google Tasks items that have a due date on its own. In the MVP, Iris writes to Google Calendar only through Google Tasks.

## Open questions

| # | Question | Owner |
|---|---|---|
| Q1 | G4 says every hook event is "eventually" ingested. What is the time bound? | Thiago |
| Q2 | What are the exit criteria for phase 4 (deployment)? | Thiago |
| Q3 | How are the measurable targets checked: G1's ≥ 99% propagation and 2-minute p95, and NFR-02's 100 ms p95? Nothing measures them today. | Thiago |
| Q4 | Event sync (FR-15): which tag marks a task as an event, which calendar it goes to, how long a timed event lasts, and what happens when the tag is added to or removed from an existing task? | Thiago |

## User stories

| ID | As the user, I want to… | Acceptance |
| --- | --- | --- |
| US-01 | run `task add "Buy milk" due:tomorrow` and see it on my phone | Item appears in the Google Tasks list with tomorrow's date (and so in Google Calendar) and in Habitica as a todo |
| US-02 | tick a task in Google Tasks and have Habitica reward me | Habitica todo is scored up; Taskwarrior on both PCs shows `completed` after their next agent run |
| US-03 | edit a task on my work PC and see it on my personal PC | Personal PC agent imports the change within one poll interval; no hook fires during import |
| US-04 | keep using `task` when Iris is down | Command succeeds with no added network delay; the event is delivered after Iris returns |
| US-05 | move a task to Friday in Google Calendar or Google Tasks | Task due becomes Friday; Taskwarrior and Habitica show Friday, and a time of day set in Taskwarrior is kept |
| US-06 | (removed) delete a calendar event | Calendar events are no longer synced in the MVP (FR-08 removed) |
| US-07 | never see a task duplicated because of sync | Echoes and replays are suppressed (`NOOP_ECHO`/`NOOP_DUPLICATE` in logs) |
| US-08 | see what failed to sync and retry it | The command center lists every sync that failed permanently, and retrying one sends it again |
| US-09 | have a task created on my phone appear in Taskwarrior | Agents on both PCs import it with the same UUID |
| US-10 | keep private tasks off other platforms | Tasks tagged `+nosync` are not sent by the hook |

## Canonical model & field support matrix

Canonical task fields: `title`, `notes`, `status` (`PENDING|COMPLETED|DELETED`), `priority` (`NONE|LOW|MEDIUM|HIGH`), `project`, `tags[]`, `dueAt` + `dueAllDay`, `completedAt`.

| Canonical | Taskwarrior | Google Tasks | Habitica (todo) |
| --- | --- | --- | --- |
| title | `description` | `title` | `text` |
| notes | — | `notes` | `notes` |
| status | `status` (pending/waiting→PENDING) | `status` + `deleted` | `completed` / deletion |
| priority | `priority` H/M/L | — | `priority` 0.1/1/1.5/2 |
| project | `project` | — | — |
| tags | `tags` | — | — |
| due | `due` (full) | `due` (date only) | `date` (date only) |

"—" means the platform neither sends nor overwrites that field (ADR-006).
