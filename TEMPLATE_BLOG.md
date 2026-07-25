# Deploy and Host NocoDB Self-Hosted on Railway

NocoDB is the open-source Airtable alternative that turns your database into a smart spreadsheet. Self-host on Railway to keep your data under your control, avoid SaaS vendor lock-in, and get started in under five minutes — with a full four-service stack (app, background worker, Redis, and PostgreSQL) provisioned automatically.

## About Hosting NocoDB Self-Hosted

Airtable's pricing scales with your headcount, not your usage — a five-person team on Airtable Business pays $1,200 a year for what is, underneath the polish, a set of database tables. NocoDB gives you the same spreadsheet-over-database experience without the per-seat tax, because you're paying for the compute you actually use on Railway, not a subscription tier gated by feature flags.

The bigger reason teams choose self-hosting over Airtable isn't the money, though — it's where the data lives. Client records, internal operations data, anything under an NDA or compliance requirement: with NocoDB self-hosted, that data sits in a Postgres database you control, not a third-party's multi-tenant cluster.

NocoDB's real differentiator versus most "spreadsheet UI over a database" tools is that it isn't limited to one database engine. It connects to PostgreSQL, MySQL, SQLite, MariaDB, SQL Server, and other ODBC-compatible databases — so if your team already has data living in an existing production database, NocoDB can put a collaborative UI on top of it directly, rather than requiring a data migration into a new proprietary format first.

## What This Template Actually Deploys

Most "NocoDB on Railway" guides only cover a two-service setup: the app plus Postgres. That works for light usage, but breaks down the moment your team runs a genuinely large CSV import or export — that operation runs on the same process serving everyone's live requests, visibly slowing the app for everyone else at the same time.

This template deploys the real four-service architecture NocoDB's own official Docker Compose reference uses: the main app, a dedicated **worker** (the identical image, running with `NC_WORKER_CONTAINER=true`) handling background jobs off the request path, **Redis** for job-state coordination, and managed **PostgreSQL** for metadata. Redis isn't a nice-to-have — without it, the worker has no way to know what jobs are pending.

## Common Use Cases

- **Customer and CRM databases** — Client records, deal pipelines, and interaction history without hand-rolling a backend.
- **Internal operations tracking** — Inventory, task boards, and cross-team workflows kept in sync in real time.
- **Feedback and bug tracking** — Collaborative tables for feature requests and issue triage, visible to your whole team at once.
- **Content and editorial pipelines** — Organize drafts, media, and publishing status with automated status transitions.
- **Generated APIs for other tools** — Every table gets REST and GraphQL endpoints automatically, useful for wiring NocoDB into Zapier, n8n, or a custom app without writing backend code.
- **Bulk data operations** — Large imports and exports route through the dedicated worker, so they don't degrade the experience for everyone else in the app at the same time.

## Dependencies for NocoDB Self-Hosted Hosting

### Deployment Dependencies

NocoDB needs PostgreSQL for its metadata (table schemas, views, filters, user accounts) and Redis for caching plus background-job coordination between the app and worker services. This template provisions both as managed Railway services, connected entirely over Railway's private network.

### Implementation Details

The template deploys `nocodb/nocodb:2026.07.0` — a specific, verified release tag, not a floating `:latest` — on port 8080 for the app, with the identical image in worker mode for the second service. `NC_DB` uses NocoDB's own connection-string format (`pg://host:port?u=user&p=password&d=database`), genuinely different from a standard Postgres URL — using the wrong format is a common way self-hosted NocoDB setups quietly fail to start. `NC_REDIS_URL` links both services to the same Redis instance, and `NC_AUTH_JWT_SECRET` is auto-generated once and shared identically between them.

## How NocoDB compares to the alternatives

**Vs. Airtable** — Airtable is polished and requires zero infrastructure knowledge, but charges per user and keeps your data on their servers. NocoDB trades a small amount of that polish for full data ownership and zero per-seat cost, which matters a lot once a team grows past a handful of people.

**Vs. Supabase** — Supabase is a genuinely strong developer-first backend platform, but it's Postgres-only and API-first — you're expected to write code to build on top of it. NocoDB is UI-first: internal teams and non-engineers can build and manage tables directly, and it works across multiple database engines instead of locking you into one.

**Vs. Baserow** — Both are open-source Airtable alternatives, but NocoDB has a considerably larger community (40,000+ GitHub stars), a more mature feature set, and generally handles larger datasets (100k+ rows) more comfortably than Baserow does.

## Getting Started

After deploying, wait for all four services to come online — the app depends on Postgres and Redis being reachable first, so give it a minute past the point where Postgres itself shows healthy. Open your Railway domain, and NocoDB walks you through creating the first admin account right there in the browser; there's no pre-seeded default login to change here, unlike some other self-hosted tools.

From there, click "Add Table" and either import an existing CSV or start from a blank spreadsheet. Column types, relations between tables, and views (grid, gallery, kanban, calendar) are all configured through the UI. If you're migrating from Airtable specifically, NocoDB's CSV import handles most of the transition directly — you generally don't need an intermediate export/reformat step.

Once your first table exists, check the API tab: NocoDB auto-generates a full REST API and a GraphQL schema for every table you create, with API tokens scoped per base. This is what makes NocoDB genuinely useful as a lightweight backend, not just a spreadsheet replacement — you can point Zapier, n8n, or a custom script at those generated endpoints without writing any server code yourself.

One thing worth knowing upfront: this template's worker and app services don't share local disk storage (a real Railway limitation — you can't mount one volume across two separate services the way single-host Docker Compose can). For most usage this is invisible, since your table data lives in Postgres either way. It only matters for attachment-heavy workflows where the worker needs a file the app just saved locally — configuring S3-compatible object storage instead of local disk sidesteps it entirely.

## Why Deploy NocoDB Self-Hosted on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying NocoDB self-hosted on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.

## Frequently Asked Questions

### Why does this template need a separate worker and Redis, if other NocoDB guides don't use them?
Many simpler guides deploy just the app plus Postgres, which works fine for light usage. But without a worker, background jobs (bulk imports, large exports, scheduled automations) run on the same process handling everyone's live requests — a big job can visibly slow the app down while it runs. This template matches NocoDB's own official reference architecture instead, which keeps those jobs off the request path entirely.

### Do I need to configure the worker myself?
No — it's provisioned and wired to the app and Redis automatically. The only thing that makes it a worker instead of a second copy of the app is `NC_WORKER_CONTAINER=true`, already set.

### What happens if Redis goes down?
Unlike some optional-Redis setups, NocoDB's worker genuinely depends on Redis to receive job assignments. If Redis is unreachable, background jobs will stop processing until it recovers — though the core app (browsing tables, editing records) keeps working normally, since that doesn't route through the worker.

### Can I connect NocoDB to a database other than Postgres for my actual tables?
Yes — the Postgres in this template only stores NocoDB's own metadata. Your actual data tables can connect to MySQL, SQLite, MariaDB, SQL Server, or other Postgres instances entirely, configured through NocoDB's own UI after deployment.

### Is NocoDB really free, or is there a catch at scale?
The self-hosted core is AGPL-3.0 open source and free forever, with no row limits, user limits, or feature gates. NocoDB's business model is a separate managed cloud offering and enterprise support contracts — self-hosting on Railway means you never touch that tier unless you choose to.

### How is this different from just running NocoDB's Docker Compose file myself on a VPS?
Functionally, very similar — this template mirrors NocoDB's own official four-service reference architecture. The difference is operational: Railway handles TLS certificates, private networking between services, managed Postgres backups, and horizontal scaling without you managing a VPS, Docker daemon, or reverse proxy configuration by hand.

### Will my existing Airtable data transfer over?
NocoDB supports direct CSV import for individual tables, covering most straightforward migrations. Complex Airtable bases with heavy linked records, lookups, or automations may need manual reconstruction after import, since those structures don't map one-to-one between platforms.
