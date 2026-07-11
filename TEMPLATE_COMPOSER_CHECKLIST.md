# Railway Template Composer Checklist — NocoDB v0.263.1

Apply these settings in the Railway template composer when generating the template from the project.

---

## 1. Healthcheck Settings

### nocodb (App Service)
- **Healthcheck Path:** `/`
- **Healthcheck Timeout:** `120` seconds (allows time for Node.js startup)
- **Variable:** `RAILWAY_HEALTHCHECK_PATH` = `/` with description

---

## 2. Variable Descriptions (Add to EVERY variable)

### nocodb Service Variables

| Variable | Description | Default / Reference |
|----------|-------------|---------------------|
| `PORT` | The port the NocoDB application listens on. | `8080` |
| `NODE_ENV` | Node.js environment mode. | `production` |
| `RAILWAY_HEALTHCHECK_PATH` | Endpoint Railway uses to verify the NocoDB service is healthy. | `/` |
| `NC_DB` | Database connection string for NocoDB's meta database. Points to the Postgres service. | `pg://${{Postgres.PGUSER}}:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/${{Postgres.PGDATABASE}}` |
| `NC_AUTH_JWT_SECRET` | Secret key for JWT session encryption and authentication. Auto-generated per deployment. | `${{secret(32)}}` |
| `NC_ATTACHMENT_FIELD_SIZE_LIMIT` | Maximum file size in MB for attachment uploads. | `20` |
| `NC_MAX_REQUEST_SIZE_MB` | Maximum request body size in MB. | `50` |

### postgres Service Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `POSTGRES_USER` | Username for the Postgres superuser account. | `postgres` |
| `POSTGRES_PASSWORD` | Password for the Postgres superuser. Auto-generated per deployment. | `${{secret(16)}}` |
| `POSTGRES_DB` | Default database name created on startup. | `nocodb` |
| `PGDATA` | Directory where PostgreSQL stores its data files inside the container. | `/var/lib/postgresql/data/pgdata` |
| `PGPORT` | Port the Postgres database listens on. | `5432` |
| `DATABASE_URL` | Auto-generated connection string from Railway. | Auto-set |

---

## 3. Auto-Injected Variables — Default Values

For the **postgres** service, Railway auto-injects these variables. Set their defaults so users don't get "needs configuration" prompts:

| Variable | Default Value | Mark Optional? | Applies To |
|----------|---------------|----------------|------------|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Yes | Docker + Managed |
| `PGPORT` | `5432` | Yes | Docker + Managed |
| `POSTGRES_DB` | `nocodb` | Yes | Docker + Managed |
| `POSTGRES_USER` | `postgres` | Yes | Docker + Managed |
| `SSL_CERT_DAYS` | `820` | Yes | Managed Postgres only |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Yes | Managed Postgres only |

---

## 4. Secrets That Must Use `${{secret()}}`

**NEVER** hardcode real passwords from the dev project. Use these template functions:

| Variable | Template Syntax |
|----------|-----------------|
| `POSTGRES_PASSWORD` | `${{secret(16)}}` |
| `NC_AUTH_JWT_SECRET` | `${{secret(32)}}` |

---

## 5. NC_DB Connection String

The `NC_DB` variable must reference the Postgres service variables to form a valid connection string. Use this format in the template composer:

```
pg://${{Postgres.PGUSER}}:${{Postgres.PGPASSWORD}}@${{Postgres.PGHOST}}:${{Postgres.PGPORT}}/${{Postgres.PGDATABASE}}
```

This ensures the NocoDB app automatically connects to the Postgres service without requiring manual configuration.

---

## 6. Optional Variables

NocoDB has no required optional variables for a basic deployment. All environment variables have sensible defaults or are auto-generated.

---

## 7. Post-Deploy Steps

After the template is published, test-deploy from a fresh Railway account (incognito window) to verify:

1. No "needs configuration" prompts appear for `PGDATA`, `PGPORT`, `POSTGRES_DB`, etc.
2. The NocoDB service comes online within 2 minutes.
3. The app responds with HTTP 200 at `/` (the root path).
4. You can access the NocoDB UI and create an admin account on first visit.
5. The Postgres database is accessible from the NocoDB app and tables can be created.

---

## 8. Verification Checklist

Before publishing to Railway marketplace:

- [ ] Healthcheck path is set to `/` with 120-second timeout
- [ ] `RAILWAY_HEALTHCHECK_PATH` variable is present with description
- [ ] `NC_DB` variable uses proper Postgres service reference syntax
- [ ] `NC_AUTH_JWT_SECRET` uses `${{secret(32)}}` template syntax
- [ ] All PGDATA, PGPORT, POSTGRES_DB variables have default values set
- [ ] All auto-injected Postgres variables are marked as "optional"
- [ ] Test deploy works without configuration prompts
- [ ] NocoDB app is healthy within 2 minutes after deploy
- [ ] Can create admin account and build tables via UI
