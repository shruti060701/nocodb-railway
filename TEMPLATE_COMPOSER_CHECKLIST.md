# Railway Template Composer Checklist — NocoDB

Apply these settings in the Railway template composer when generating the template from the project. Rewritten 2026-07-25 — the previous version was for a stale, never-GitHub-connected 2-service setup (`nocodb:0.263.1` + Postgres only) and had two real bugs: a wrong `startCommand` (`node server.js`, which doesn't exist in the NocoDB image — the real entrypoint is `node docker/main.js`) and a wrong `NC_DB` connection-string format (a standard `pg://user:pass@host:port/db` URL, when NocoDB actually expects a query-string format: `pg://host:port?u=user&p=pass&d=db`, confirmed against NocoDB's own real GitHub `docker-compose/examples/quickstart-demo/docker-compose.yml`).

**Real services this template deploys:** `nocodb` (the app), `nocodb-worker` (same image, background-job mode), `redis`, `Postgres`.

---

## 1. Healthcheck Settings

### `nocodb` (app service)
- **Healthcheck Path:** `/api/v1/health` — this is NocoDB's real documented health endpoint (confirmed via the vendor's own `docker-compose.yml`, which healthchecks the app with `wget ... http://localhost:8080/api/v1/health`). **Not `/`** — the previous checklist version had this wrong.
- **Healthcheck Timeout:** `120` seconds

### `nocodb-worker`
- **Leave the Healthcheck Path blank.** The vendor's own compose file does not healthcheck the worker container at all — only the main `nocodb` service. Setting an HTTP healthcheck here risks the exact bug already caught and fixed on this project's Umami template (`valkey` failing forever because its healthcheck path was set to an HTTP path on a service that doesn't reliably serve one). Leave this as a TCP-only check, or disabled entirely if Railway's UI allows it for a service with no exposed public domain.

### `redis` / `Postgres`
- No public port exposed — no healthcheck needed (internal services only)

---

## 2. Variable Descriptions (Add to EVERY variable)

### `nocodb` (App) Variables

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `PORT` | `8080` | No | The port NocoDB listens on internally. |
| `NODE_ENV` | `production` | No | Node.js runtime environment mode. |
| `NC_DB` | `pg://${{Postgres.PGHOST}}:${{Postgres.PGPORT}}?u=${{Postgres.PGUSER}}&p=${{Postgres.PGPASSWORD}}&d=${{Postgres.PGDATABASE}}` | No | Connection string for NocoDB's meta database (table schemas, views, user accounts). **Uses NocoDB's own query-string format, not a standard Postgres URL** — verified against the vendor's real docker-compose example. Getting this format wrong is a silent failure mode worth flagging clearly to anyone editing this later. |
| `NC_AUTH_JWT_SECRET` | `${{secret(32)}}` | No | Secret key for JWT token generation and session validation. Auto-generated per deployment. |
| `NC_REDIS_URL` | `redis://${{redis.RAILWAY_PRIVATE_DOMAIN}}:6379` | No | Redis connection used for caching and coordinating background jobs with the worker service. **Required for the worker to function** — unlike Umami's Redis, which is optional with a graceful fallback, NocoDB's worker/app split genuinely needs this to communicate. |
| `NC_SITE_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | No | Public-facing URL, used for invitation links and password-reset emails. Uses Railway's dynamic domain reference so it resolves correctly for every future deployer, not just this test instance. |

### `nocodb-worker` Variables

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `NC_DB` | `${{nocodb.NC_DB}}` | No | Must be the exact same meta-database connection as the main `nocodb` service — reference it directly rather than duplicating the string, so the two never drift apart. |
| `NC_AUTH_JWT_SECRET` | `${{nocodb.NC_AUTH_JWT_SECRET}}` | No | Must match the main app's secret exactly, or sessions/tokens won't validate consistently between the two. Cross-service reference, not an independent `${{secret()}}` generation — this exact "independently generated instead of referenced" mistake has already happened once on this project (Typebot's `ENCRYPTION_SECRET`), so it's called out explicitly here. |
| `NC_REDIS_URL` | `${{nocodb.NC_REDIS_URL}}` | No | Same reasoning — must reference the app's value, not generate its own. |
| `NC_SITE_URL` | `${{nocodb.NC_SITE_URL}}` | No | Used for consistency in generated links; the worker itself has no public domain. |
| `NC_WORKER_CONTAINER` | `true` | No | The setting that actually turns this instance into a background-job worker instead of a second copy of the web app. This is the one variable genuinely unique to this service. |

### `redis` Variables

No custom variables needed — runs on image defaults (`redis:8.2.1`, matches the version used in Railway's own official reference template, verified to exist on Docker Hub). Only referenced by `nocodb`/`nocodb-worker` via `RAILWAY_PRIVATE_DOMAIN`.

### `Postgres` Variables (managed plugin — `railwayapp-templates/postgres-ssl`)

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `DATABASE_URL` | Auto-set by Railway's plugin — leave as is | No | Standard connection string. Not directly used by NocoDB (which needs the `NC_DB` query-string format instead), but other tools/clients may still expect it. |
| `DATABASE_PUBLIC_URL` | Auto-set by Railway's plugin — leave as is | No | Public/external connection string for reaching this database from outside Railway's network (e.g. a local GUI client). |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | No | Internal hostname — this is what `NC_DB` actually connects through. |
| `PGPORT` | `5432` | No | Port Postgres listens on internally. Verify this is actually filled in, not left as an empty "to be filled by the user" placeholder — this exact composer glitch has been seen on this project's Umami template. |
| `PGUSER` | `${{POSTGRES_USER}}` | No | Database username. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | No | Database name. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | No | Database password. |
| `POSTGRES_USER` | `postgres` | **Yes** | Username for the Postgres superuser account. |
| `POSTGRES_PASSWORD` | Whatever Railway's plugin actually prefills — **verify live via the composer screenshot, don't assume a specific `secret()` length**. This exact wrong guess (assuming `secret(16)` instead of checking) has already happened on two other templates in this project (Evolution API, Typebot). | No | Auto-generated superuser password. |
| `POSTGRES_DB` | `railway` (Railway's own default) | **Yes** | Default database name created on startup. |
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | **Yes** | Directory where Postgres stores its data files. |
| `SSL_CERT_DAYS` | `820` | **Yes** | SSL certificate validity period. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | **Yes** | Seconds Railway waits for active connections before a redeploy. Verify this is actually filled in, same empty-placeholder caveat as `PGPORT` above. |

---

## 3. Secrets That Must Use `${{secret()}}`

**NEVER** hardcode real credentials from the dev project.

| Variable | Template Syntax |
|----------|-----------------|
| `NC_AUTH_JWT_SECRET` (on `nocodb`) | `${{secret(32)}}` |
| `NC_AUTH_JWT_SECRET` (on `nocodb-worker`) | `${{nocodb.NC_AUTH_JWT_SECRET}}` — cross-reference, NOT an independent `secret()` call |
| `POSTGRES_PASSWORD` | Whatever Railway's plugin already prefilled — verify live, don't assume a length |

---

## 4. Volumes

- Mount a Railway Volume on the `nocodb` service only, at `/usr/app/data` (matches the vendor's own compose file mount path). **Do not attempt to mount the same volume on `nocodb-worker`** — Railway doesn't support sharing one volume across two services the way Docker Compose does. This is a known, documented limitation of this template (see README's Notes section), not something to try to work around.

---

## 5. Known Troubleshooting

- **NocoDB fails to start / can't reach the database:** almost always means `NC_DB` is using the wrong string format. It is NOT a standard `postgresql://user:pass@host:port/db` URL — NocoDB requires `pg://host:port?u=user&p=pass&d=db`. Double-check this first if `nocodb` crash-loops right after Postgres comes online.
- **Background jobs never complete (imports, exports, scheduled automations hang forever):** check that `NC_REDIS_URL` is set identically (via cross-reference, not independent value) on both `nocodb` and `nocodb-worker`, and that the `redis` service is actually running.
- **Worker healthcheck failing / stuck in "Deploying":** the worker should have no healthcheck path set at all. If one got set (e.g. copied from the app service's settings), clear it — this is the same bug class already caught on this project's Umami/Valkey pairing.
- **Uploaded files/attachments not visible from certain operations:** expected, given the documented volume-sharing limitation above — not a bug to chase, a known architectural gap worth calling out to users if they report it.

---

## 6. Post-Deploy Steps

After the template is published, test-deploy from a fresh Railway account (incognito window) and verify:

1. No "needs configuration" prompts appear for Postgres's auto-injected variables (`PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE`, `PGPASSWORD`, `PGDATA`, `SSL_CERT_DAYS`, `RAILWAY_DEPLOYMENT_DRAINING_SECONDS`).
2. All four services (`nocodb`, `nocodb-worker`, `redis`, `Postgres`) come online — check `nocodb-worker` specifically, since it's the most likely to be silently missed or misconfigured given it's new to this template.
3. The app responds with a real `200` at `/api/v1/health`.
4. Open the actual Railway domain in a browser and create a real admin account — don't just curl the root path.
5. Create a table, then test a background-job-dependent action (e.g. a bulk CSV import) to confirm the worker is actually processing jobs, not just that the service shows "Running."
