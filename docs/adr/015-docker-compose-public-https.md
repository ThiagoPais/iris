---
type: adr
project: iris
number: 015
status: accepted
updated: 2026-11-16
---

# 015. Deployment: single container + Postgres, reachable over public HTTPS

## Context

Google Apps Script and Habitica webhooks must reach Iris from the internet.

## Decision

Deploy with Docker Compose (`iris-hub` + `postgres:17`) on a VPS behind a TLS reverse proxy (Caddy). A home host exposed through Cloudflare Tunnel or Tailscale Funnel is an equivalent alternative. Migrations run on container start (`prisma migrate deploy`).

## Alternatives considered

TODO: not recorded at decision time.

## Consequences

A purely local, unexposed daemon is not supported while Google or Habitica inbound is enabled.
