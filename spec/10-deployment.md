# 10 — Deployment & operations

**Goal:** a reproducible production stack on a VPS with TLS, migrations on start, backups and a runbook.
**ADRs:** 015, 012.

## 1. `Dockerfile` (multi-stage, `node:24-alpine`)

1. `deps`: `npm ci`.
2. `build`: copy the sources, `npx prisma generate`, `npm run build`, `npm prune --omit=dev`. **Keep `prisma`** (the CLI is needed for `migrate deploy`): move it to `dependencies`, or install it separately in the runtime stage.
3. `runtime`: non-root `node` user; copy `dist/`, `node_modules/`, `prisma/`, `prisma.config.ts` (if present), `package.json`. `ENV NODE_ENV=production`. `HEALTHCHECK` = `wget -qO- http://127.0.0.1:3000/health || exit 1`.
4. Entrypoint `docker/entrypoint.sh`: `npx prisma migrate deploy && exec node dist/main.js`.

Add `.dockerignore` (`node_modules`, `dist`, `.env*`, `agents`, `docs`, `spec`, `.git`).

## 2. `docker-compose.prod.yml`

- `postgres:17-alpine`: volume `iris_pg`, **no published port**, healthcheck.
- `iris-hub`: `build: .`, `env_file: .env`, `depends_on: postgres (service_healthy)`, `expose: 3000` only, `restart: unless-stopped`.
- `caddy:2-alpine`: ports `80`, `443`, volumes `./Caddyfile`, `caddy_data`, `caddy_config`.

`Caddyfile`:
```text
{$IRIS_DOMAIN} {
  encode zstd gzip
  reverse_proxy iris-hub:3000
  log {
    output stdout
    format filter {
      wrap json
      fields { request>uri regexp "^/v1/ingest/habitica/[^/?]+" "/v1/ingest/habitica/REDACTED" }
    }
  }
}
```

Check the `format filter` syntax against the Caddy 2 docs you have; if it has changed, keep the same redaction and log the change in `spec/QUESTIONS.md`.

## 3. Backups (`scripts/backup.sh`)

`docker compose -f docker-compose.prod.yml exec -T postgres pg_dump -U iris -Fc iris > backups/iris-$(date +%F).dump`, then delete dumps older than 14 days. Document a host crontab line (`0 3 * * *`) and the restore command (`pg_restore --clean --if-exists`).

## 4. Runbook (`docs/runbook.md`) — the agent writes it with these sections, commands only, no prose padding

1. **Server:** DNS → VPS, clone, `cp .env.example .env`, generate secrets (`openssl rand -base64 48 | tr -d '/+=' | cut -c1-48`) for the 3 tokens + Habitica secret, set `IRIS_TIMEZONE`, `IRIS_DOMAIN`, `docker compose -f docker-compose.prod.yml up -d --build`, verify `curl https://$IRIS_DOMAIN/health`.
2. **Google:** create an OAuth Desktop client, enable the Tasks and Calendar APIs, `npm run google:auth` locally, set the `GOOGLE_*` values and the target IDs (how to find the task list ID via the API Explorer `tasklists.list`; the calendar ID from Calendar settings), add targets to `IRIS_PUSH_TARGETS`, restart.
3. **Apps Script:** as in `integrations/google-apps-script/README.md`.
4. **Habitica:** set `HABITICA_USER_ID`/`HABITICA_API_KEY` (Settings → API), `npm run habitica:webhook https://$IRIS_DOMAIN`.
5. **Each PC:** `agents/taskwarrior/install.sh`, edit `agent.json` (URL, agent token, unique `deviceId`), `iris_agent.py push-all` (**on one PC only**), `iris_agent.py status`.
6. **Operations:** health, `GET /v1/sync/jobs?status=DEAD`, retry, token rotation (edit `.env` → restart → update clients), restore from backup.

## Tests

- `docker build .` succeeds; `docker compose -f docker-compose.prod.yml config` validates.
- Smoke script `scripts/smoke.sh <base> <adminToken>`: `/health` 200, `GET /v1/tasks` 200 with the admin token, 401 without.

## Done when

The stack comes up locally with `IRIS_DOMAIN=localhost` (Caddy internal TLS), and the smoke script passes.
