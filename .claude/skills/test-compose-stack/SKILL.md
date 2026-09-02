---
name: test-compose-stack
description: Bring up and end-to-end test the self-hosted Elmo stack from the checked-in docker-compose.yml (postgres + db-migrate + web + worker), including bootstrap registration and brand onboarding.
---

# Testing the self-hosted docker compose stack

## Bring it up

```bash
cd <repo root>
cp .env.example .env   # placeholders are enough for UI-only testing
docker compose up -d --build   # first build takes several minutes on a 2-CPU box
```

Compose project name is `elmo`, so containers are `elmo-web-1`, `elmo-worker-1`,
`elmo-postgres-1`, `elmo-db-migrate-1`. Web is published on `${WEB_PORT:-1515}`.

Healthy state: `docker compose ps -a` shows postgres `Up (healthy)`, web `Up`, worker `Up`,
and `db-migrate` `Exited (0)` (it is a one-shot migration job — its absence from plain
`docker compose ps` is normal, use `-a`).

Worker readiness: `docker compose logs --tail=20 worker` should end with
`All handlers registered, worker is ready`. Also check `docker inspect -f '{{.RestartCount}}'`
on web/worker to rule out a crash loop that logs alone can hide.

## Walking the UI

`DEPLOYMENT_MODE=local` means:

- `http://localhost:1515` 307-redirects to `/auth/register` until the single bootstrap
  signup exists (`apps/web/src/routes/auth/register.tsx`). Local mode does **not** require
  email verification and does not offer Google OAuth, so any email/password works.
- After "Create account" you land on `/app`, and a `Default` organization is
  auto-provisioned — there is no separate org-creation step to click through.
- `Create your first brand` → `/app/org/default/new` (Brand Name + Website) →
  `/app/org/default/brand/<slug>` which shows the "Research Brand Data" /
  "Analyze brand" onboarding step.

## What placeholder keys do and don't break

With placeholder `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `DATAFORSEO_*`, env validation
passes and everything up to and including brand creation works. Clicking **Analyze brand**
is expected to show `Brand analysis failed. Please try again.` — that is not a stack bug.

Useful proof that web → pg-boss → worker wiring is intact even without real keys: after
clicking Analyze brand, `docker compose logs worker` should show
`[onboarding] analyzeBrand start: <url>`. If that line never appears, the job queue
plumbing (not the AI keys) is broken.

## Persistence check

`docker compose restart web`, wait ~10s, reload the browser. You should stay logged in and
still see the brand you created. If you land back on `/auth/register`, the postgres volume
(`postgres_data` mounted at `/var/lib/postgresql`, not the PG18 version subdirectory) is not
persisting.

## Env parity check

Compare key names only — `.env` may legitimately carry `DEPLOYMENT_ID` (a random UUID the
`elmo` CLI writes for telemetry; it is not needed when `DISABLE_TELEMETRY=1`), so its absence
from `.env.example` is not necessarily a defect:

```bash
for k in $(grep -oE '^[A-Z_]+' .env | sort -u); do grep -q "^$k=" .env.example || echo "missing: $k"; done
```

## Devin Secrets Needed

None for stack/UI testing. Real `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `DATAFORSEO_LOGIN`,
and `DATAFORSEO_PASSWORD` are required only to exercise brand analysis and prompt runs.
