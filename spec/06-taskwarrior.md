# 06 — Taskwarrior (hub side + hook + agent)

**Goal:** Taskwarrior task **creation** (`on-add`) and **modification** (`on-modify`) reach Iris without slowing the CLI, and changes from elsewhere land in Taskwarrior on every PC.
**ADRs:** 008, 009, 012, 014. **Docs:** [PRD: field support matrix](../docs/iris-prd.md#canonical-model--field-support-matrix) (TW column).

## Part A — Hub

### A1. Mapper (`src/platforms/taskwarrior/taskwarrior.mapper.ts`, pure)

Taskwarrior date format: `YYYYMMDDTHHMMSSZ` (UTC). Helpers: `parseTwDate(s): Date`, `formatTwDate(d): string`.

`toInbound(ev: { event: 'add'|'modify'; task: TwTask }, { tz, deviceId })`:
- `status`: `recurring` → `ignore('recurrence template')`. `pending|waiting` → `PENDING`, `completed` → `COMPLETED`, `deleted` → `DELETED`. Anything else → `ignore('unknown status')`.
- `tags` contains `nosync` → `ignore('nosync')`.
- `externalId = task.uuid`; `op = 'UPSERT'` (TW deletions are status changes).
- `fields` (**all six keys always set**): `title = description`; `status`; `priority`: `H/M/L` → `HIGH/MEDIUM/LOW`, missing → `NONE`; `project ?? null`; `tags ?? []`; `due = task.due ? fromInstant(parseTwDate(task.due), tz) : null`.
- `sourceUpdatedAt = parseTwDate(task.modified ?? task.entry)`; `completedAt = task.end ? parseTwDate(task.end) : null`.
- `linkExtra = { entry: task.entry }`; `deviceId` from ctx.

`toTw(task, link, tz): TwTaskOut` for the feed. **Every key is present**; `null` means "remove locally":
```ts
{ uuid: link.externalId, description: title, status: 'pending'|'completed'|'deleted',
  entry: (link.extra.entry as string) ?? formatTwDate(task.createdAt),
  modified: formatTwDate(task.lastEditAt),
  end: status==='PENDING' ? null : formatTwDate(task.completedAt ?? task.lastEditAt),
  due: !due ? null : formatTwDate(due.allDay ? localMidnightUtc(dueDate(due,tz), tz) : due.at),
  tags: tags.length ? tags : null, priority: {HIGH:'H',MEDIUM:'M',LOW:'L',NONE:null}[p], project }
```

### A2. Endpoints (`@RequirePrincipal('agent')`)

`POST /v1/ingest/taskwarrior`
```ts
body = z.object({
  deviceId: z.string().regex(/^[A-Za-z0-9._-]{1,64}$/),
  events: z.array(z.object({
    event: z.enum(['add','modify']),
    task: z.object({ uuid: z.string().uuid(), description: z.string() }).passthrough(),
    spooledAt: z.string().datetime(),
  })).min(1).max(500),
})
→ 200 { results: Array<{ uuid, outcome, taskId }> }   // order = input order
```
Map each event (ignore → `IGNORED`, logged in SyncLog with `taskId null`), then `ingestMany`. Strip nothing (TW payloads hold no secrets).

`GET /v1/agents/taskwarrior/changes?deviceId=&since=0&limit=200` → `changesSince('TASKWARRIOR', …)` → `200 { cursor: string, tasks: TwTaskOut[] }`. `since` is a decimal string parsed as `BigInt` (400 if invalid).

### A3. Hub tests

- Mapper unit tests: every status; `due:tomorrow` (local midnight) → all-day; timed due; `nosync`; recurring; round-trip `toInbound(toTw(x))` preserves fields.
- E2E: device A POSTs add → device B feed returns it with the same uuid; device A feed does not; B POSTs modify (done) → A's feed shows `completed` with `end`.

## Part B — Local scripts (`agents/taskwarrior/`, Python ≥ 3.11, **stdlib only**)

### B1. Files

| File | Role |
| --- | --- |
| `iris_common.py` | config, paths, spool append/read/trim with `fcntl` locks, HTTP helper (`urllib.request`), logging to `agent.log` (rotate at 1 MB, keep 1) |
| `iris_hook.py` | hook entrypoint (symlinked twice) |
| `iris_agent.py` | CLI: `flush`, `pull`, `sync`, `push-all`, `status` |
| `install.sh` | idempotent installer |
| `systemd/iris-agent.service`, `systemd/iris-agent.timer` | user units |
| `tests/test_hook.py`, `tests/test_agent.py` | unittest |

Paths: config `${XDG_CONFIG_HOME:-~/.config}/iris/agent.json` = `{ "url", "token", "deviceId", "taskBin": "task" }`. State `${XDG_STATE_HOME:-~/.local/state}/iris/`: `spool.jsonl`, `spool.lock`, `flush.lock`, `cursor`, `rejected.jsonl`, `agent.log`.

### B2. `iris_hook.py` — MUST

1. Mode = `add` if `basename(sys.argv[0])` starts with `on-add`, else `modify`.
2. Read all stdin. The last non-empty line is `out` (the task to echo): `add` has 1 line, `modify` has 2 (original, modified).
3. In a `finally` block: write `out + "\n"` to stdout, flush, `sys.exit(0)`. **No other stdout output. Never a non-zero exit** (I4).
4. Inside `try` (any exception → log to `agent.log`, swallow):
   - `IRIS_SYNC_DISABLE=1` → skip.
   - Missing config → skip silently.
   - `status == 'recurring'` or `'nosync' in tags` → skip.
   - `modify`: if the original and modified tasks are equal on `description,status,due,tags,priority,project` → skip.
   - Append `{"event": mode, "task": <modified task>, "spooledAt": <UTC ISO>}` to the spool (exclusive lock, one line, `fsync`).
   - Unless `IRIS_HOOK_NO_SPAWN=1`: `subprocess.Popen([sys.executable, <realdir>/iris_agent.py, "flush"], stdin/stdout/stderr=DEVNULL, start_new_session=True, close_fds=True)`. Do not wait.
5. Resolve sibling modules via `os.path.realpath(__file__)` (the hooks are symlinks).
6. Budget: ≤ 100 ms wall time on a warm cache. No network I/O.

### B3. `iris_agent.py` — commands

- **`flush`**: take `flush.lock` non-blocking (held → exit 0). Loop: read up to 200 spool lines (corrupt JSON lines → append to `rejected.jsonl` and drop) → `POST /v1/ingest/taskwarrior` (timeout 10 s).
  - 200 → under the spool lock, remove exactly those leading lines (write the rest to a temp file, `os.replace`).
  - 400 → move the batch to `rejected.jsonl`, remove it, continue.
  - 401/403/network/5xx → log, exit 1 (the spool is kept).
- **`pull`**: `since = cursor file or "0"`. Loop: `GET /v1/agents/taskwarrior/changes?deviceId&since&limit=200` → `apply(tasks)` → write the cursor atomically → repeat while `len(tasks) == 200`.
- **`apply(tasks)`**:
  1. Export the local copies in one call: `task rc.hooks=off rc.verbose=nothing <uuid…> export` (JSON array).
  2. For each remote `r`:
     - Local missing and `r.status == 'deleted'` → skip.
     - Local present and `parse(local.modified) > parse(r.modified)` → skip (the local edit is newer and already on its way through the spool).
     - `merged = dict(local or {})`. For each key of `description,status,due,tags,priority,project,end,modified`: `None` → `pop`, else set. `uuid` from `r`; `entry` = `local.entry` if present, else `r.entry`. If `local.status == 'waiting'` and `r.status == 'pending'` → keep `waiting`.
  3. One call: `task rc.hooks=off rc.confirmation=off rc.verbose=nothing import` with the JSON array on stdin. A non-zero exit → raise (the cursor is not advanced).
- **`sync`**: `flush` (ignore its failure), then `pull`. This is what the timer runs.
- **`push-all`**: `task rc.hooks=off status:pending export` → spool one `add` event per task (same filters as the hook) → `flush`.
- **`status`**: print the device ID, URL, spool line count, cursor, and the last 5 log lines. **Never** print the token.

### B4. `install.sh`

Idempotent, `set -euo pipefail`:
1. Copy the `.py` files to `~/.local/share/iris/`.
2. Hooks directory = `task _get rc.hooks.location`, falling back to `$(task _get rc.data.location)/hooks`. `mkdir -p`, then `ln -sf ~/.local/share/iris/iris_hook.py <hooks>/on-add.iris` and `…/on-modify.iris`. `chmod +x` the target.
3. If the config is missing: write a template (`deviceId` = `hostname -s`) with mode `600` and print "edit it".
4. Install the user units to `~/.config/systemd/user/`, then `systemctl --user daemon-reload && systemctl --user enable --now iris-agent.timer`.

Units: service `Type=oneshot`, `ExecStart=/usr/bin/env python3 %h/.local/share/iris/iris_agent.py sync`. Timer: `OnBootSec=1min`, `OnUnitActiveSec=60s`, `Persistent=true`.

### B5. Python tests (`python3 -m unittest discover -s agents -p 'test_*.py'`)

Use a temp `HOME`/`XDG_*`, `IRIS_HOOK_NO_SPAWN=1`, a fake Iris (`http.server` in a thread) and a fake `task` binary (a Python script written to tmp and set as `taskBin`, recording argv and stdin and returning fixtures).
- Hook: `on-add` echoes stdin exactly and spools 1 record; `on-modify` echoes the **second** line; annotation-only modify is not spooled; `nosync` and recurring are not spooled; garbage stdin → still echoes and exits 0; missing config → echoes, no spool; runtime < 300 ms.
- Flush: 200 removes the sent lines only (a line appended concurrently survives); 500 keeps everything and exits 1; 400 moves the batch to rejected.
- Pull/apply: overlay removes keys set to `null`; keeps local annotations; skips when local is newer; skips unknown deleted; import is called with `rc.hooks=off`; the cursor is not advanced when import fails.
- Optional (skipped if `task` is not on PATH): a real Taskwarrior round-trip with `TASKDATA`/`TASKRC` in tmp.

## Done when

Hub lint, build, unit and e2e tests pass; the Python unittest command passes; manual test: two `TASKDATA` directories with different `deviceId`s, `task add` in one appears in the other after `iris_agent.py sync`.
