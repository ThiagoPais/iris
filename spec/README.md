# Iris — Execution Specs (for AI agents)

These files are **instructions for an AI coding agent**. Each spec is one work package, meant to be executed in order in one session or PR. The specs are normative: MUST/MUST NOT are binding, SHOULD may be deviated from only with a written reason in the PR description.

## Sources of truth (priority order)

1. [`docs/adr/`](../docs/adr/README.md): decisions. When a spec and an ADR conflict, the ADR wins; stop and log the conflict (see Protocol).
2. [`docs/iris-dbdd.md`](../docs/iris-dbdd.md): the database schema. Its [Prisma schema](../docs/iris-dbdd.md#prisma-schema) section is copied **verbatim** into `prisma/schema.prisma`.
3. [`docs/iris-prd.md`](../docs/iris-prd.md): behavior, [field support matrix](../docs/iris-prd.md#canonical-model--field-support-matrix), user stories.
4. These specs: file layout, contracts, algorithms, tests.

## Execution order

| # | Spec | Phase | Depends on |
| --- | --- | --- | --- |
| 00 | [conventions](./00-conventions.md) | — (read always) | — |
| 01 | [foundation](./01-foundation.md) | 1 | 00 |
| 02 | [auth](./02-auth.md) | 1 | 01 |
| 03 | [domain](./03-domain.md) | 1 | 01 |
| 04 | [ingest & change feed](./04-ingest.md) | 1 | 02, 03 |
| 05 | [dispatch, worker & admin API](./05-dispatch.md) | 1 | 04 |
| 06 | [taskwarrior](./06-taskwarrior.md) | 2 | 05 |
| 07 | [google tasks + apps script](./07-google-tasks.md) | 3 | 05 |
| 08 | [google calendar](./08-google-calendar.md) | post-MVP (PRD FR-15) | 07 |
| 09 | [habitica](./09-habitica.md) | 3 | 05 |
| 10 | [deployment](./10-deployment.md) | 4 | 05 (+ any adapters) |

Spec 08 is not part of the MVP: skip it until event sync (PRD FR-15, open question Q4) is designed. It must then be revised to sync only tasks marked as events, and the parts of ADR-010 it changes need a superseding ADR.

## Protocol (every spec)

1. Read `00-conventions.md`, then the target spec in full, then the ADRs it cites.
2. Check preconditions: previous specs' "Done when" still pass (`npm run lint && npm test`).
3. Implement **only** what the spec lists. Do not add features, endpoints, fields or dependencies that are not listed.
4. If the spec is ambiguous, wrong, or conflicts with an ADR or an external API's real behavior: choose the most conservative option, add an entry to `spec/QUESTIONS.md` (`## <spec> — <topic>`: context, choice made, alternatives), and continue.
5. Write the tests listed under **Tests** (more are welcome, none may be skipped).
6. Run every command under **Done when** and paste the output summary in the PR or commit body.
7. Commit with a conventional message: `feat(<scope>): <spec #> <summary>`.

## Global invariants (MUST hold after every spec)

- I1: Every canonical change is written in one DB transaction together with its `TaskChange`, `SyncLog` and `SyncJob` rows.
- I2: No code path writes to a platform without a later `ExternalLink.lastSyncedHash` update, or a `SyncLog` `FAILED` row.
- I3: The origin platform comes only from route + principal, never from the request body or headers.
- I4: Taskwarrior hooks exit `0` and echo the required JSON in all cases, including exceptions.
- I5: No secret appears in logs, `SyncLog.rawPayload`, or error messages.
- I6: `src/domain/**` imports nothing from NestJS, Prisma or Node I/O (pure functions only).
