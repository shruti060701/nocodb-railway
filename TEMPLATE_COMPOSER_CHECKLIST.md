# Railway Template Composer Checklist — NocoDB

Apply these settings in the Railway template composer when generating the template from the project. Rewritten 2026-07-25 (twice, same day) — first to move from a stale, never-GitHub-connected 2-service setup to a 4-service one matching Railway's official reference template (app + worker + Redis + Postgres), then reverted back to 2 services after live-testing found a real, reproducible bug in the 4-service setup (see "Why no worker service" below).

**Real services this template deploys:** `nocodb` (the app, single instance), `Postgres`.

---

## Why no separate worker service

NocoDB officially supports splitting background job processing (CSV imports, exports, automations) into a dedicated worker container via `NC_WORKER_CONTAINER=true`, and Railway's own official reference template (`railway.com/deploy/nocodb`) deploys exactly that: app + worker + Redis + Postgres.

**We built that 4-service version first, then live-tested it with a real CSV import — and it failed.** Root cause, confirmed via the worker's own deploy logs, not inferred:

```
CSV import failed: Failed to stream file: ENOENT: no such file or directory,
access '/usr/app/data/nc/uploads/2026/07/25/.../nocodb-test-import_0svxH.csv'
---- !! JOB FAILED !! ----
```

What happened: the app received the CSV upload and wrote it to its own local disk. The import job was then picked up by the separate worker service — a different container, with its own separate filesystem. Railway does not support mounting one Volume across two services the way Docker Compose does, so the worker has no way to see a file that only exists on the app's disk. **This isn't an edge case — it broke the exact CSV import flow shown on NocoDB's own "Getting Started" screen**, the first thing most new users try.

**Fix applied: dropped the worker (and Redis, which existed only to coordinate app/worker job state) entirely.** A single NocoDB instance processes background jobs in-process, sharing its own local disk with itself — no cross-container file access needed. Re-tested the identical CSV import against this simplified setup and confirmed it now works end-to-end: real rows landed in a real table (8/8 rows from a test CSV, verified by opening the table in a browser, not just checking the job status).

**If you want the worker-split architecture anyway** (e.g. for genuinely high background-job volume where in-process handling becomes a bottleneck), the real fix is configuring S3-compatible object storage for uploads instead of local disk, so both containers can read the same files over the network. That requires an external storage account (Cloudflare R2, Backblaze B2, etc.) and isn't part of this template, since it adds real setup friction that conflicts with a one-click deploy.

---

## 1. Healthcheck Settings

### `nocodb` (app service)
- **Healthcheck Path:** `/api/v1/health` — this is NocoDB's real documented health endpoint (confirmed via the vendor's own `docker-compose.yml`, which healthchecks with `wget ... http://localhost:8080/api/v1/health`). **Not `/`** — an earlier version of this checklist had this wrong.
- **Healthcheck Timeout:** `120` seconds

### `Postgres`
- No public port exposed — no healthcheck needed (internal service only)

---

## 2. Variable Descriptions (Add to EVERY variable)

### `nocodb` (App) Variables

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `PORT` | `8080` | No | The port NocoDB listens on internally. |
| `NODE_ENV` | `production` | No | Node.js runtime environment mode. |
| `NC_DB` | `pg://${{Postgres.PGHOST}}:${{Postgres.PGPORT}}?u=${{Postgres.PGUSER}}&p=${{Postgres.PGPASSWORD}}&d=${{Postgres.PGDATABASE}}` | No | Connection string for NocoDB's meta database (table schemas, views, user accounts). **Uses NocoDB's own query-string format, not a standard Postgres URL** — verified against the vendor's real docker-compose example, and confirmed working on a real live deploy (the app started cleanly and a real CSV import succeeded against this exact string). Getting this format wrong is a silent failure mode worth flagging clearly to anyone editing this later. |
| `NC_AUTH_JWT_SECRET` | `${{secret(32)}}` | No | Secret key for JWT token generation and session validation. Auto-generated per deployment. |
| `NC_SITE_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | No | Public-facing URL, used for invitation links and password-reset emails. Uses Railway's dynamic domain reference so it resolves correctly for every future deployer, not just this test instance. |

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
| `NC_AUTH_JWT_SECRET` | `${{secret(32)}}` |
| `POSTGRES_PASSWORD` | Whatever Railway's plugin already prefilled — verify live, don't assume a length |

---

## 4. Volumes

Mount a Railway Volume on the `nocodb` service at `/usr/app/data` — this is where uploads, CSV import files, and other local attachments live. Confirmed via a real deploy that this volume path is correct (logs showed `Mounting volume on: ...` at container start, and a real CSV import wrote/read files successfully afterward).

---

## 5. Known Troubleshooting

- **NocoDB fails to start / can't reach the database:** almost always means `NC_DB` is using the wrong string format. It is NOT a standard `postgresql://user:pass@host:port/db` URL — NocoDB requires `pg://host:port?u=user&p=pass&d=db`. Double-check this first if `nocodb` crash-loops right after Postgres comes online.
- **CSV import / file upload fails with an ENOENT error:** if you ever reintroduce a separate worker service, this will recur — see the "Why no separate worker service" section above for the full root cause and reasoning. Don't re-add a worker without also solving shared file storage (S3-compatible object storage, not local disk).
- **Import shows "0 records" right after completing:** this is expected briefly — the import runs as an async background job even within a single instance, and the UI can render before the job fully lands. Refresh after a few seconds; a real test import confirmed rows do appear (8/8 rows landed correctly within ~10 seconds of clicking Import).

---

## 6. Post-Deploy Steps

After the template is published, test-deploy from a fresh Railway account (incognito window) and verify:

1. No "needs configuration" prompts appear for Postgres's auto-injected variables (`PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE`, `PGPASSWORD`, `PGDATA`, `SSL_CERT_DAYS`, `RAILWAY_DEPLOYMENT_DRAINING_SECONDS`).
2. Both services (`nocodb`, `Postgres`) come online.
3. The app responds with a real `200` at `/api/v1/health`.
4. Open the actual Railway domain in a browser and create a real admin account — don't just curl the root path.
5. **Actually test a CSV import**, not just that the app loads — create a small test CSV, import it via Import Data → CSV, and confirm real rows land in the resulting table after a few seconds. This exact flow is what exposed the worker-architecture bug that shaped this template's final design — it's worth re-verifying on every future regeneration of this template, not assumed to still work.
