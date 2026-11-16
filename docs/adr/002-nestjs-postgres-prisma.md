---
type: adr
project: iris
number: 002
status: accepted
updated: 2026-11-16
---

# 002. NestJS 12 (ESM) + PostgreSQL + Prisma

## Context

The repository is already scaffolded with NestJS 12 as ESM (`"type": "module"`, `.js` import suffixes), Vitest, oxlint and Prettier.

## Decision

Keep that scaffold. Use PostgreSQL 17 and the latest Prisma major version. Use `$queryRaw` only where Prisma cannot express something: `FOR UPDATE SKIP LOCKED`, advisory locks and partial unique indexes. Validate environment variables with `zod` at boot.

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

ACID transactions keep ingest and outbox writes atomic, and no Redis or broker is needed. The `@nestjs/observe` wiring in the scaffold carries placeholder credentials, so it is registered only when `OBSERVE_APP_KEY` is set.
