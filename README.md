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

## How to Use

1. Click the Deploy on Railway button above
2. Connect your GitHub account and authorize Railway
3. Set the `NC_AUTH_JWT_SECRET` environment variable (auto-generated on deploy)
4. Railway automatically provisions a PostgreSQL database for you
5. Wait for the deployment to finish and open your Railway domain
6. Create your first admin account and start building

## Notes

- **Database persistence** — NocoDB metadata is stored in the PostgreSQL service. Your data is safe across restarts
- **Port** — NocoDB runs on port 8080 inside the container. Railway automatically exposes it via HTTPS
- **Authentication** — Set a strong `NC_AUTH_JWT_SECRET` to secure your instance
- **Postgres required** — This template uses PostgreSQL as the meta database. Your actual data tables can connect to other databases (MySQL, SQLite, etc.) through NocoDB's UI

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
  nocodb/nocodb:0.263.1
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
