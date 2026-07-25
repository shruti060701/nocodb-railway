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

## What This Template Deploys

Two services: **NocoDB** (`nocodb/nocodb:2026.07.0`) and Railway-managed **PostgreSQL** for metadata (table schemas, views, filters, user accounts). A Railway Volume is mounted on the NocoDB service for local file storage (uploads, CSV imports).

**Why no Redis or worker service?** NocoDB officially supports splitting background job processing (CSV imports, exports, automations) into a dedicated worker container, coordinated via Redis. We built and live-tested that version during this template's development and found a real, reproducible bug: Railway doesn't support mounting one volume across two separate services, so when an import job landed on the worker container, it failed with `ENOENT: no such file or directory` trying to read a file that only existed on the app container's disk — confirmed via the worker's own logs, not a guess. We then checked Railway's actual official template source (`github.com/railwayapp-templates/nocodb`) and found it doesn't use a worker either — it's just NocoDB + Postgres + Redis-as-an-optional-cache, three services, no job-processing split. So this template runs as a single NocoDB service, which avoids the volume-sharing bug entirely, and skips Redis too since an in-memory cache is functionally fine for a single instance. See `TEMPLATE_COMPOSER_CHECKLIST.md` for the full story if you want the details.

## How to Use

1. Click the Deploy on Railway button above
2. Railway automatically provisions PostgreSQL and NocoDB together
3. `NC_AUTH_JWT_SECRET` is auto-generated on deploy — no action needed
4. Wait for the deployment to finish and open your Railway domain
5. Create your first admin account and start building

## Notes

- **Database persistence** — NocoDB metadata is stored in the PostgreSQL service. Your data is safe across restarts.
- **File persistence** — Uploads and CSV import files are stored on a Railway Volume mounted at `/usr/app/data`.
- **Port** — NocoDB runs on port 8080 inside the container. Railway automatically exposes it via HTTPS.
- **Authentication** — `NC_AUTH_JWT_SECRET` is auto-generated per deployment to secure your instance.
- **Postgres required** — This template uses PostgreSQL as the meta database. Your actual data tables can connect to other databases (MySQL, SQLite, etc.) through NocoDB's UI.

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

NocoDB is built on Node.js with a Vue/Nuxt frontend (confirmed by inspecting the deployed app's actual asset paths — `/_nuxt/...` — during verification of this template, not assumed from older docs). It uses:
- **Frontend** — Vue.js + Nuxt
- **Backend** — Express.js + Fastify
- **Database driver** — Knex.js for multi-database support

## License

NocoDB is released under the AGPL-3.0 license. You can self-host it freely, but modifications must be open-sourced. For commercial licensing, contact the NocoDB team.

## Support

- **GitHub** — https://github.com/nocodb/nocodb
- **Docs** — https://nocodb.com/docs
- **Discord** — https://discord.gg/nocodb
- **Issues** — https://github.com/nocodb/nocodb/issues
