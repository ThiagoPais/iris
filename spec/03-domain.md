# 03 — Domain (pure functions)

**Goal:** a framework-free core that normalizes, projects, hashes and merges task fields. Everything else builds on it.
**ADRs:** 004, 005, 006, 013, 014. **Docs:** PRD §5.
**Constraint:** `src/domain/**` imports only `node:crypto` and `luxon` (invariant I6).

## 1. Types (`src/domain/types.ts`)

```ts
export const PLATFORMS = ['TASKWARRIOR','GOOGLE_TASKS','GOOGLE_CALENDAR','HABITICA'] as const;
export type Platform = typeof PLATFORMS[number];
export const PUSH_PLATFORMS = ['GOOGLE_TASKS','GOOGLE_CALENDAR','HABITICA'] as const;
export type PushPlatform = typeof PUSH_PLATFORMS[number];
export const AGENT_PLATFORMS = ['TASKWARRIOR'] as const;
export type AgentPlatform = typeof AGENT_PLATFORMS[number];

export type TaskStatus = 'PENDING' | 'COMPLETED' | 'DELETED';
export type Priority = 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH';

export interface Due { at: Date; allDay: boolean }

export interface CanonicalFields {
  title: string;
  notes: string | null;
  status: TaskStatus;
  priority: Priority;
  project: string | null;
  tags: string[];
  due: Due | null;
}

export type FieldName = keyof CanonicalFields;   // 'title'|'notes'|'status'|'priority'|'project'|'tags'|'due'
export type FieldMode = 'full' | 'date' | 'none';
export type ProjectionTarget = Platform | 'CANONICAL';
```

These string values MUST equal the Prisma enum values, so they are assignable in both directions. Add a type-level test (`expectTypeOf`) against the generated Prisma enums in `src/sync/`, not in domain.

## 2. Field support (`src/domain/field-support.ts`)

```ts
export const FIELD_SUPPORT: Record<ProjectionTarget, Record<FieldName, FieldMode>> = {
  CANONICAL:       { title:'full', notes:'full', status:'full', priority:'full', project:'full', tags:'full', due:'full' },
  TASKWARRIOR:     { title:'full', notes:'none', status:'full', priority:'full', project:'full', tags:'full', due:'full' },
  GOOGLE_TASKS:    { title:'full', notes:'full', status:'full', priority:'none', project:'none', tags:'none', due:'date' },
  GOOGLE_CALENDAR: { title:'full', notes:'full', status:'none', priority:'none', project:'none', tags:'none', due:'full' },
  HABITICA:        { title:'full', notes:'full', status:'full', priority:'full', project:'none', tags:'none', due:'date' },
};
```

## 3. Time (`src/domain/time.ts`, luxon)

| Function | Contract |
| --- | --- |
| `toLocalDate(at: Date, tz: string): string` | `YYYY-MM-DD` of `at` in `tz` |
| `allDayDue(date: string): Due` | `{ at: new Date(date + 'T00:00:00.000Z'), allDay: true }`; throws `DomainValidationError` if not a valid ISO date |
| `dueDate(due: Due, tz: string): string` | all-day → UTC date part of `due.at`; timed → `toLocalDate(due.at, tz)` |
| `isLocalMidnight(at: Date, tz: string): boolean` | `at` in `tz` has hour, minute, second and ms all 0 |
| `localMidnightUtc(date: string, tz: string): Date` | UTC instant of `date 00:00` in `tz` |
| `fromInstant(at: Date, tz: string): Due` | `isLocalMidnight` → `allDayDue(toLocalDate(at, tz))`, else `{ at, allDay: false }` |

## 4. Normalize (`src/domain/normalize.ts`)

`normalizeFields(p: Partial<CanonicalFields>): Partial<CanonicalFields>`. Only keys present in `p` are returned. For each present key:
- `title`: trim, collapse internal `\r\n` → `\n`; if empty → throw `DomainValidationError('title required')`; truncate to 1024 chars.
- `notes`: trim; `''` → `null`.
- `project`: trim; `''` → `null`.
- `tags`: trim each, lowercase, drop empty, dedupe, sort ascending.
- `due`: if `allDay`, force `at` to UTC midnight of its UTC date; milliseconds kept for timed dues.
- `status`, `priority`: validated against the unions.

## 5. Projection & hash (`src/domain/projection.ts`, `src/domain/hash.ts`)

`project(target: ProjectionTarget, f: CanonicalFields, tz: string): Record<string, unknown>`. Output contains only fields whose mode ≠ `none`:
- `title`, `notes`, `status`, `priority`, `project`: as-is.
- `tags`: as-is (already sorted).
- `due` with mode `full`: `null` or `{ at: due.at.toISOString(), allDay }`.
- `due` with mode `date`: `null` or `dueDate(due, tz)`.

`stableStringify(v)`: JSON with object keys sorted recursively; arrays keep order; `undefined` keys omitted.
`hashProjection(target, f, tz) = sha256hex(stableStringify(project(target, f, tz)))` → 64 lowercase hex chars.
`contentHash(f, tz) = hashProjection('CANONICAL', f, tz)`.

## 6. Defaults & deletion candidate (`src/domain/merge.ts`)

```ts
export const DEFAULT_FIELDS: Omit<CanonicalFields,'title'> =
  { notes: null, status: 'PENDING', priority: 'NONE', project: null, tags: [], due: null };

// State that a platform's "item deleted" represents (ADR-013).
export function deletionCandidate(platform: Platform, f: CanonicalFields): CanonicalFields {
  return platform === 'GOOGLE_CALENDAR' ? { ...f, due: null } : { ...f, status: 'DELETED' };
}

// Whether a push platform should hold a remote item for this task.
export function shouldExistRemotely(platform: PushPlatform, f: CanonicalFields): boolean {
  if (f.status === 'DELETED') return false;
  if (platform === 'GOOGLE_CALENDAR') return f.due !== null;
  return true;
}
```

## 7. Merge (`src/domain/merge.ts`)

```ts
export function mergeInbound(
  current: CanonicalFields | null,
  incoming: Partial<CanonicalFields>,    // already normalized
  platform: Platform,
  tz: string,
): CanonicalFields
```

Rules, in order:
1. `base = current ?? { ...DEFAULT_FIELDS, title: undefined }`.
2. For each key `k` **present** in `incoming` (`k in incoming`) with `FIELD_SUPPORT[platform][k] !== 'none'`:
   - mode `full` → `result[k] = incoming[k]`.
   - mode `date` (only `due`): if `incoming.due === null` → `null`; else if `base.due !== null && dueDate(base.due, tz) === dueDate(incoming.due, tz)` → keep `base.due`; else `incoming.due` (must be all-day).
3. Keys absent from `incoming`, or unsupported by the platform → `base[k]`.
4. If `result.title` is undefined → throw `DomainValidationError('title required')`.
5. Return a new object; never mutate inputs.

`completedAt` is **not** a domain field. `IngestService` derives it (spec 04).

## Tests (`*.spec.ts`, table-driven)

- `time`: São Paulo (UTC−3) — `2026-10-01T03:00:00Z` is local midnight → all-day `2026-10-01`; `2026-10-01T15:00:00Z` → timed; `localMidnightUtc('2026-10-01','America/Sao_Paulo')` = `2026-10-01T03:00:00.000Z`.
- `normalize`: tag ordering and dedup; empty title throws; `''` notes → null.
- `projection`: GOOGLE_TASKS excludes tags/priority/project; a timed due projects to the local date; CALENDAR excludes status.
- `hash`: key order does not matter; identical fields → identical hash; a tags-only change alters CANONICAL and TASKWARRIOR hashes but not GOOGLE_TASKS.
- `merge`:
  - GOOGLE_TASKS incoming without tags keeps current tags.
  - GOOGLE_TASKS same date keeps the timed due (`14:00` retained).
  - GOOGLE_TASKS different date replaces it with all-day.
  - TASKWARRIOR `notes` in incoming is ignored.
  - `incoming.due = null` clears the due.
  - A new task without a title throws.
- `deletionCandidate`, `shouldExistRemotely`: every branch.

## Done when

`npm test` passes; `src/domain` has ≥ 95% line coverage (`npm run test:cov`); `grep -r "@nestjs\|generated/prisma" src/domain` returns nothing.
