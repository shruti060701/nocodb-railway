# NocoDB — Open-Source Airtable Alternative & No-Code Database Platform

Deploy NocoDB, the open-source alternative to Airtable, on Railway with one click.

## Deploy on Railway

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/new/template)

## Features

- **Open-source database UI** — Spreadsheet-like interface for PostgreSQL, MySQL, SQLite, and more
- **No-code workflows** — Build automations, API endpoints, and data pipelines without coding
- **API-first design** — Full REST & GraphQL APIs for custom integrations
- **Team collaboration** — Real-time sync, comments, and role-based access control
- **Self-hosted control** — Your data lives on your infrastructure, not a SaaS vendor's servers
- **Airtable-compatible** — Familiar workflows for teams migrating from Airtable
- **Background job processing** — A dedicated worker service handles imports, exports, and automations off the request path, so large jobs don't slow down the app your team is actively using

## What This Template Deploys

This template provisions four services together:

1. **NocoDB** (`nocodb/nocodb:2026.07.0`) — the main app and web UI, public-facing
2. **NocoDB Worker** — the same image, running in background-job mode (`NC_WORKER_CONTAINER=true`), no public port
3. **Redis** — required for both the app and worker to share cache and job-queue state; without it, the two containers can't coordinate background jobs correctly
4. **PostgreSQL** (Railway's managed Postgres) — stores NocoDB's metadata (table schemas, views, filters, user accounts)

## How to Use

1. Click the Deploy on Railway button above
2. Railway automatically provisions PostgreSQL, Redis, the NocoDB app, and the NocoDB worker together
3. `NC_AUTH_JWT_SECRET` is auto-generated on deploy — no action needed
4. Wait for all four services to come online (the app service depends on Postgres and Redis being reachable)
5. Open your Railway domain and create your first admin account

## Notes

- **Database persistence** — NocoDB metadata is stored in the PostgreSQL service. Your data is safe across restarts.
- **Port** — NocoDB runs on port 8080 inside the container. Railway automatically exposes it via HTTPS.
- **Authentication** — `NC_AUTH_JWT_SECRET` is auto-generated per deployment to secure your instance.
- **Postgres required** — This template uses PostgreSQL as the meta database. Your actual data tables can connect to other databases (MySQL, SQLite, etc.) through NocoDB's UI.
- **Redis required for the worker to function** — Without `NC_REDIS_URL` set correctly on both the app and worker, background jobs (bulk imports, large exports, scheduled automations) will not process.
- **Known limitation — local attachment storage isn't shared between the app and worker** — the official NocoDB Docker Compose setup mounts one shared volume across both containers; Railway doesn't support mounting a single volume across two separate services. This template only mounts a volume on the main `nocodb` service. For most usage (data stored in Postgres, attachments served via a table's UI) this doesn't matter — but if you're running attachment-heavy workflows that rely on the worker processing locally-uploaded files, consider configuring S3-compatible storage instead of local disk.

## Self-Hosting on Other Platforms

Clone the repository:
```bash
git clone https://github.com/nocodb/nocodb
cd nocodb
```

For Docker Compose:
```bash
docker run -d --name noco -v "$(pwd)"/nocodb:/usr/app/data/ -p 8080:8080 \
  -e NC_DB="pg://host:5432?u=postgres&p=password&d=nocodb" \
  -e NC_AUTH_JWT_SECRET="your-secret-key" \
  nocodb/nocodb:2026.07.0
```

For Kubernetes:
```bash
helm repo add nocodb https://charts.nocodb.com
helm install nocodb nocodb/nocodb
```

## Architecture

NocoDB is built on Node.js with a React frontend. It uses:
- **Frontend** — React + TypeScript
- **Backend** — Express.js + Fastify
- **Database driver** — Knex.js for multi-database support
- **ORM** — Prisma (for NocoDB's meta database)

## License

NocoDB is released under the AGPL-3.0 license. You can self-host it freely, but modifications must be open-sourced. For commercial licensing, contact the NocoDB team.

## Support

- **GitHub** — https://github.com/nocodb/nocodb
- **Docs** — https://nocodb.com/docs
- **Discord** — https://discord.gg/nocodb
- **Issues** — https://github.com/nocodb/nocodb/issues
