# Deploy and Host NocoDB Self-Hosted on Railway

NocoDB is the open-source Airtable alternative that turns your database into a smart spreadsheet. Self-host on Railway to keep your data under your control, avoid SaaS vendor lock-in, and get started in under five minutes.

## About Hosting NocoDB Self-Hosted

Airtable's pricing scales with your headcount, not your usage — a five-person team on Airtable Business pays $1,200 a year for what is, underneath the polish, a set of database tables. NocoDB gives you the same spreadsheet-over-database experience without the per-seat tax, because you're paying for the compute you actually use on Railway, not a subscription tier gated by feature flags.

The bigger reason teams choose self-hosting isn't the money, though — it's where the data lives. Client records, internal operations data, anything under an NDA or compliance requirement: with NocoDB self-hosted, that data sits in a Postgres database you control, not a third-party's multi-tenant cluster.

NocoDB's real differentiator versus most "spreadsheet UI over a database" tools is that it isn't limited to one database engine. It connects to PostgreSQL, MySQL, SQLite, MariaDB, SQL Server, and other ODBC-compatible databases — so if your team already has data in an existing production database, NocoDB can put a collaborative UI on top of it directly, rather than requiring a migration into a new proprietary format first.

## Why This Template Doesn't Use a Separate Worker Service (and why the split quietly breaks on Railway anyway)

NocoDB supports splitting background job processing (CSV imports, large exports, scheduled automations) into a second container, running the identical image with `NC_WORKER_CONTAINER=true`, coordinated via Redis. On a single host running Docker Compose, this works cleanly, because both containers can mount the same shared volume.

We built and live-tested that exact 4-service version (app, worker, Redis, Postgres) with a real CSV import before publishing anything. It failed. Railway doesn't support mounting one Volume across two services, so the worker had no way to see the file the app had just written to its own local disk. The worker's own logs showed the real error plainly:

```
CSV import failed: Failed to stream file: ENOENT: no such file or directory
---- !! JOB FAILED !! ----
```

This isn't a rare edge case — it broke the exact CSV import flow on NocoDB's own "Getting Started" screen, the first thing most new users try. We then checked Railway's actual official template source directly (`github.com/railwayapp-templates/nocodb`, not just its marketing page) and found it doesn't use a worker at all — it's just the app, Postgres, and Redis as an optional cache, three services total. So this template goes a step further and runs as a single instance with no Redis either, since an in-memory cache is functionally fine without a worker to coordinate with. Background jobs process in-process against the same local disk uploads are written to, so there's no cross-container file access to break. We re-ran the identical import against this setup and confirmed all rows landed correctly, verified by opening the resulting table in a browser.

## System Requirements for Self-Hosting NocoDB

NocoDB is comfortable on modest hardware for most teams. A minimum viable setup is 1 shared vCPU, 1 GB RAM, and 10 GB of storage — enough for a small team's internal tools and a few thousand rows across tables. If you're managing 10,000+ rows, running frequent large CSV imports, or supporting a larger team hitting the app concurrently, bump up to 2 vCPU and 4 GB RAM. Storage needs scale mostly with attachments (images, files) rather than raw table data, since Postgres handles structured data efficiently regardless of row count.

## Common Use Cases

- **Customer and CRM databases** — Client records, deal pipelines, and interaction history without hand-rolling a backend.
- **Internal operations tracking** — Inventory, task boards, and cross-team workflows kept in sync in real time.
- **Feedback and bug tracking** — Collaborative tables for feature requests and issue triage, visible to your whole team at once.
- **Content and editorial pipelines** — Organize drafts, media, and publishing status with automated status transitions.
- **Generated APIs for other tools** — Every table gets REST and GraphQL endpoints automatically, useful for wiring NocoDB into Zapier, n8n, or a custom app without writing backend code.

## Dependencies for NocoDB Self-Hosted Hosting

### Deployment Dependencies

NocoDB needs PostgreSQL for its metadata (table schemas, views, filters, user accounts). This template provisions it as a managed Railway service, connected entirely over Railway's private network, plus a Railway Volume for local file storage.

### Implementation Details

The template deploys `nocodb/nocodb:2026.07.0` — a specific, verified release tag, not a floating `:latest` — on port 8080. `NC_DB` uses NocoDB's own connection-string format (`pg://host:port?u=user&p=password&d=database`), genuinely different from a standard Postgres URL — using the wrong format is a common way self-hosted NocoDB setups quietly fail to start. `NC_AUTH_JWT_SECRET` is auto-generated per deployment for session security.

## How NocoDB compares to the alternatives

**Vs. Airtable** — Airtable is polished and requires zero infrastructure knowledge, but charges per user and keeps your data on their servers. NocoDB trades a small amount of that polish for full data ownership and zero per-seat cost, which matters a lot once a team grows past a handful of people.

**Vs. Supabase** — Supabase is a genuinely strong developer-first backend platform, but it's Postgres-only and API-first — you're expected to write code to build on top of it. NocoDB is UI-first: internal teams and non-engineers can build and manage tables directly, and it works across multiple database engines instead of locking you into one.

**Vs. Baserow** — Both are open-source Airtable alternatives, but NocoDB has a considerably larger community (40,000+ GitHub stars), a more mature feature set, and generally handles larger datasets (100k+ rows) more comfortably than Baserow does.

## Getting Started

After deploying, open your Railway domain, and NocoDB walks you through creating the first admin account right there in the browser — there's no pre-seeded default login to change here, unlike some other self-hosted tools.

From there, click "Add Table" and either import an existing CSV or start from a blank spreadsheet. Column types, relations between tables, and views (grid, gallery, kanban, calendar) are all configured through the UI. If you're migrating from Airtable specifically, NocoDB's CSV import handles most of the transition directly — you generally don't need an intermediate export/reformat step.

Once your first table exists, check the API tab: NocoDB auto-generates a full REST API and a GraphQL schema for every table you create, with API tokens scoped per base. This is what makes NocoDB genuinely useful as a lightweight backend, not just a spreadsheet replacement — you can point Zapier, n8n, or a custom script at those generated endpoints without writing any server code yourself.

## Why Deploy NocoDB Self-Hosted on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying NocoDB self-hosted on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.

## Frequently Asked Questions

### Does this match Railway's own official NocoDB template?
Almost — the real official template (`railwayapp-templates/nocodb` on GitHub, verified directly rather than trusting its marketing page) is 3 services: app, Postgres, and Redis as an optional cache. No worker. This template goes one step simpler, dropping Redis too, since without a worker there's nothing for it to coordinate — NocoDB just falls back to in-memory caching, which is functionally fine for a single instance.

### Will background jobs slow down the app for other users?
Not meaningfully for typical usage — jobs run asynchronously even on one instance. For genuinely high job volume, the correct fix is S3-compatible object storage plus a separate worker, not local-disk sharing.

### Can I connect NocoDB to a database other than Postgres for my actual tables?
Yes — the Postgres in this template only stores NocoDB's own metadata. Your actual data tables can connect to MySQL, SQLite, MariaDB, SQL Server, or other Postgres instances entirely, configured through NocoDB's own UI after deployment.

### Is NocoDB really free, or is there a catch at scale?
The self-hosted core is AGPL-3.0 open source and free forever, with no row limits, user limits, or feature gates. NocoDB's business model is a separate managed cloud offering and enterprise support contracts — self-hosting on Railway means you never touch that tier unless you choose to.

### Will my existing Airtable data transfer over?
NocoDB supports direct CSV import for individual tables, covering most straightforward migrations. Complex Airtable bases with heavy linked records, lookups, or automations may need manual reconstruction after import, since those structures don't map one-to-one between platforms.

### How was this template actually verified before publishing?
Beyond a healthy deployment status, we drove a real browser through the full flow: created an admin account, uploaded a test CSV through Import Data, and confirmed every row landed in the resulting table. That real end-to-end test is what caught the worker bug above — a passing healthcheck alone wouldn't have surfaced it.
