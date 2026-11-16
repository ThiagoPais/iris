# 00 — Conventions

Applies to every spec. Read it before any other spec.

## Repository layout (target)

```text
iris-hub/
├─ prisma/                    schema.prisma, migrations/
├─ prisma.config.ts           (Prisma ≥ 7)
├─ src/
│  ├─ main.ts, app.module.ts
│  ├─ config/                 env.schema.ts, config.module.ts
│  ├─ prisma/                 prisma.service.ts, prisma.module.ts
│  ├─ generated/prisma/       (generated; git-ignored)
│  ├─ common/                 zod-validation.pipe.ts, errors.ts, json-logger.ts
│  ├─ auth/                   principal.ts, auth.guard.ts, decorators.ts
│  ├─ domain/                 PURE: types, normalize, projection, hash, merge, time
│  ├─ sync/                   ingest.service.ts, change-feed.service.ts, outbox.service.ts,
│  │                          sync-worker.service.ts, push-adapter.ts, maintenance.service.ts
│  ├─ platforms/
│  │  ├─ taskwarrior/         mapper, controller, dto
│  │  ├─ google/              google-auth.service.ts, tasks/, calendar/, controller
│  │  └─ habitica/            client, adapter, mapper, controller
│  ├─ admin/                  admin.controller.ts, admin.service.ts
│  └─ health/                 health.controller.ts
├─ agents/
│  └─ taskwarrior/            iris_hook.py, iris_agent.py, install.sh, systemd/, tests/
├─ integrations/google-apps-script/   Code.gs, appsscript.json, README.md
├─ scripts/                   google-auth.ts, habitica-webhook.ts, backup.sh
├─ test/                      *.e2e-spec.ts, helpers/
├─ docker-compose.yml, Dockerfile, Caddyfile, .env.example
```

## TypeScript / NestJS

- ESM: relative imports **MUST** end in `.js` (e.g. `import { X } from './x.js'`). `"type": "module"` stays.
- Strict TS; no `any` in `src/domain` or `src/sync`. `unknown` + zod parse at every boundary.
- Validation: **zod** only (no class-validator). Controllers use `ZodValidationPipe(schema)` from `src/common`.
- Dependency injection: constructor injection. Adapters are registered under the multi-provider token `PUSH_ADAPTERS`.
- Errors: throw `DomainValidationError` (domain), `RetryableError`, `PermanentError` or `RemoteNotFoundError` (adapters) from `src/common/errors.ts`. Controllers never leak stack traces.
- Time: `Date` in UTC everywhere. Timezone logic only via `src/domain/time.ts` (luxon).
- IDs: `crypto.randomUUID()`.
- Logging: Nest `Logger` with the JSON console logger; include `taskId`, `platform`, `outcome` where relevant.
- Formatting: Prettier (single quotes, trailing commas). Lint: `npm run lint` must be clean.
- No new runtime dependencies beyond those listed in the specs. Allowed set: `@prisma/client`, `prisma`, `@prisma/adapter-pg`, `pg`, `zod`, `luxon`, `@googleapis/tasks`, `@googleapis/calendar`, `google-auth-library`, `@nestjs/schedule`. Allowed dev dependencies: `prisma`, `dotenv-cli`, `@types/pg`, `@types/luxon`.

## Tests

- Unit: `*.spec.ts` beside the source, run with `npm test` (Vitest, globals).
- E2E: `test/*.e2e-spec.ts`, run with `npm run test:e2e` against a real Postgres (`DATABASE_URL` pointing at the `iris_test` DB). Each e2e file truncates all tables in `beforeEach` (`TRUNCATE … RESTART IDENTITY CASCADE`).
- External HTTP is **never** called in tests. Adapters receive an injectable client; tests use fakes. Google clients are wrapped behind small interfaces for this.
- Python: `python3 -m unittest discover -s agents -p 'test_*.py'`, standard library only.

## Naming

- Platform enum values: `TASKWARRIOR | GOOGLE_TASKS | GOOGLE_CALENDAR | HABITICA`.
- Push targets (outbox): `GOOGLE_TASKS | GOOGLE_CALENDAR | HABITICA`. Agent platforms (feed): `TASKWARRIOR`. Echoing platforms = push targets.
- HTTP routes are versioned `/v1/...`, except `GET /health`.

## Definition of Done (every spec)

- `npm run lint`, `npm run build`, `npm test` pass. `npm run test:e2e` passes when the spec touches the DB.
- Python specs: the unittest command passes.
- No TODOs left in code for listed requirements.
- `.env.example` updated with any new variable (with comment).
