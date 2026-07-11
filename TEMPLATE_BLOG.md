# Deploy and Host NocoDB self-hosted on Railway

NocoDB is the open-source Airtable alternative that turns your database into a smart spreadsheet. Self-host on Railway to keep your data under your control, avoid SaaS vendor lock-in, and get started in under five minutes.

## About Hosting NocoDB self-hosted

Hosting your own instance means your data stays private and on infrastructure you own. NocoDB's no-code interface lets teams build database tables, automations, and APIs without writing SQL or backend code. Railway simplifies the hosting side by providing managed PostgreSQL, automatic HTTPS, and zero-configuration networking. You avoid managing servers, backups, and SSL certificates while keeping 100% control of your data.

## Common Use Cases

- Internal team workflows and CRM systems where you need full control over data
- Customer-facing data tables and portals that require privacy and customization
- Rapid prototyping of database applications without writing backend code
- Data collection forms and surveys that feed into structured tables
- Automated workflows that sync data between NocoDB and external tools

## Dependencies for NocoDB self-hosted Hosting

NocoDB is a Node.js application that requires PostgreSQL for storing table schemas, user accounts, and automations. The database can be any SQL database NocoDB supports.

### Deployment Dependencies

- **PostgreSQL 12+** — Meta database for NocoDB's internal configuration
- **Node.js 18+** — Runtime environment (included in the Docker image)
- **Docker** — Containerization (optional for this Railway template)

### Implementation Details

NocoDB uses the official Docker image `nocodb/nocodb:0.263.1` running on port 8080. The NC_DB environment variable specifies the PostgreSQL connection string. NC_AUTH_JWT_SECRET handles session encryption and should be set to a strong random value. Railway's private DNS connects the app to Postgres without exposing the database to the internet.

## Why Deploy NocoDB self-hosted on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying NocoDB self-hosted on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
