# Railway Template Composer Checklist — Plane

Apply these settings in the Railway template composer when generating the template from the project. Built 2026-09-06 against `makeplane/plane-aio-community:v1.4.2` — current stable per Docker Hub at build time. Pinned to an explicit version rather than the moving `:stable` tag (the reference marketplace template uses `:stable`, but an explicit version avoids an untracked breaking change landing silently on every future deploy).

**Real, live service names in this project:**

| Real Service Name | What It Actually Is |
|---|---|
| `plane` | The Plane all-in-one app (web, API, workers, live server behind Caddy) — `makeplane/plane-aio-community:v1.4.2` |
| `postgres` | Primary datastore — `ghcr.io/railwayapp-templates/postgres-ssl:18` |
| `redis` | Cache/session store — `redis:8.2.1` |
| `rabbitmq` | Background task queue — `rabbitmq:3.13.6-management-alpine` |
| `minio` | S3-compatible object storage for uploads — `minio/minio:latest` |

---

## 1. Healthcheck Settings

### `plane`
- No HTTP healthcheck path set — leave blank. First boot runs Postgres migrations across multiple internal processes (web/api/worker/live) behind Caddy; a healthcheck that fires before migrations finish would false-flag a good deploy as failed. TCP-only check is the safer default here given the multi-process boot sequence.
- If health score regresses in practice, `/api/instances/` is a real unauthenticated JSON endpoint that returns instance info once the API server is ready — usable as a healthcheck path if this needs revisiting.

### `postgres`, `redis`, `rabbitmq`, `minio`
- No healthcheck path — these are internal-only backing services with no public domain, matching standard practice for supporting infrastructure.

---

## 2. Custom Start Commands

| Service | Start Command | Why |
|---|---|---|
| `redis` | `redis-server --requirepass $REDIS_PASSWORD --appendonly yes` | Official redis image has no default auth; without `--requirepass` the cache is open on the private network. `--appendonly yes` persists data to the mounted volume across restarts. |
| `minio` | `minio server /data --console-address :9001` | The bare `minio/minio` image exits immediately with no args — confirmed live (first deploy attempt exited in 0s with no start command set). |

---

## 3. Variable Descriptions (Add to EVERY variable)

### `plane` — 12 total

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `DATABASE_URL` | `postgresql://plane:${{postgres.POSTGRES_PASSWORD}}@${{postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/plane` | No | Full Postgres connection string. Must reference the `postgres` service's own generated password — never hardcode. |
| `REDIS_URL` | `redis://default:${{redis.REDIS_PASSWORD}}@${{redis.RAILWAY_PRIVATE_DOMAIN}}:6379` | No | Redis connection string used for caching and session storage. Password must match `redis`'s `REDIS_PASSWORD`. |
| `AMQP_URL` | `amqp://plane:${{rabbitmq.RABBITMQ_DEFAULT_PASS}}@${{rabbitmq.RAILWAY_PRIVATE_DOMAIN}}:5672/plane` | No | RabbitMQ connection string for background task processing (notifications, imports, exports). Vhost must be `plane`, matching `RABBITMQ_DEFAULT_VHOST` on the `rabbitmq` service. |
| `AWS_REGION` | `us-east-1` | No | Required by the S3 client even though MinIO ignores real AWS regions — any valid-looking region string works. |
| `AWS_ACCESS_KEY_ID` | `${{minio.MINIO_ROOT_USER}}` | No | Must match `minio`'s `MINIO_ROOT_USER` — this is the object storage access key, not a real AWS credential. |
| `AWS_SECRET_ACCESS_KEY` | `${{minio.MINIO_ROOT_PASSWORD}}` | No | Must match `minio`'s `MINIO_ROOT_PASSWORD`. |
| `AWS_S3_BUCKET_NAME` | `uploads` | No | Bucket name for file/attachment uploads. Plane's backend creates this bucket on first use if it doesn't exist — confirmed live, no manual `mc mb` step needed. |
| `AWS_S3_ENDPOINT_URL` | `http://${{minio.RAILWAY_PRIVATE_DOMAIN}}:9000` | No | **Critical** — without this, the app defaults to real `s3.amazonaws.com` and every upload silently fails auth against MinIO credentials. |
| `APP_PROTOCOL` | `https` | No | Railway domains are always HTTPS; this must be `https`, not the container's internal default of `http`, or generated links/CORS will be wrong. |
| `SECRET_KEY` | `${{secret(50)}}` | No | Django cryptographic secret for sessions and tokens. Auto-generated per deployment if left as the shipped placeholder, but setting an explicit long random value here at publish time is safer than relying on first-boot autogeneration. |
| `LIVE_SERVER_SECRET_KEY` | `${{secret(50)}}` | No | Shared secret between the API and the real-time collaboration server. Same autogeneration caveat as `SECRET_KEY`. |
| `DOMAIN_NAME` | `${{RAILWAY_PUBLIC_DOMAIN}}` | No | **Bare hostname only, no `https://` prefix.** Confirmed live — a protocol prefix here breaks the container's own domain validation regex and the app refuses to boot. |

### `postgres` — 4 total

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `POSTGRES_USER` | `plane` | No | Must match the username used in `plane`'s `DATABASE_URL`. |
| `POSTGRES_PASSWORD` | `${{secret(32)}}` | No | Auto-generated Postgres password — referenced by `plane`'s `DATABASE_URL`. |
| `POSTGRES_DB` | `plane` | No | Database name — must match the path segment in `plane`'s `DATABASE_URL`. |
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | No | Data directory inside the mounted volume. |

### `redis` — 1 total

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `REDIS_PASSWORD` | `${{secret(32)}}` | No | Used by the custom start command's `--requirepass` flag and referenced in `plane`'s `REDIS_URL`. |

### `rabbitmq` — 3 total

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `RABBITMQ_DEFAULT_USER` | `plane` | No | Must match the username in `plane`'s `AMQP_URL`. |
| `RABBITMQ_DEFAULT_PASS` | `${{secret(32)}}` | No | Referenced by `plane`'s `AMQP_URL`. |
| `RABBITMQ_DEFAULT_VHOST` | `plane` | No | Must match the vhost path segment in `plane`'s `AMQP_URL`. |

### `minio` — 2 total

| Variable | Value | Mark Optional? | Description |
|----------|-------|-----------------|-------------|
| `MINIO_ROOT_USER` | `${{secret(16)}}` | No | Object storage access key — referenced by `plane`'s `AWS_ACCESS_KEY_ID`. |
| `MINIO_ROOT_PASSWORD` | `${{secret(32)}}` | No | Object storage secret key — referenced by `plane`'s `AWS_SECRET_ACCESS_KEY`. |

---

## 4. Secrets That Must Use `${{secret()}}`

| Variable | Service | Template Syntax |
|----------|---------|-----------------|
| `POSTGRES_PASSWORD` | postgres | `${{secret(32)}}` |
| `REDIS_PASSWORD` | redis | `${{secret(32)}}` |
| `RABBITMQ_DEFAULT_PASS` | rabbitmq | `${{secret(32)}}` |
| `MINIO_ROOT_USER` | minio | `${{secret(16)}}` |
| `MINIO_ROOT_PASSWORD` | minio | `${{secret(32)}}` |
| `SECRET_KEY` | plane | `${{secret(50)}}` |
| `LIVE_SERVER_SECRET_KEY` | plane | `${{secret(50)}}` |

---

## 5. Volumes

| Service | Mount Path | Notes |
|---------|-----------|-------|
| `postgres` | `/var/lib/postgresql/data` | All workspace/project/issue data. Without this, every redeploy wipes the entire instance. |
| `redis` | `/data` | Persists the append-only file (`--appendonly yes`) so cached sessions survive restarts — not critical data, but avoids forcing every user to re-login after a redeploy. |
| `minio` | `/data` | All uploaded files and attachments. Without this, every redeploy wipes every uploaded file. |

---

## 6. Known Troubleshooting

- **`/api/instances/` (or any API route) 502s right after deploy:** expected during first boot — confirmed live during this template's own test deploy. The AIO image runs Postgres migrations across multiple supervised processes before the API server starts accepting connections. The web UI (static assets via Caddy) responds with 200 well before the API does — don't mistake that for "fully ready."
- **API returns 500 with `redis.exceptions.AuthenticationError: invalid username-password pair or user is disabled` in logs, even though the Redis password variable looks correct:** confirmed live during this template's own build — `railway redeploy` / the dashboard's plain "Redeploy" button reuses the OLD deployment's baked-in start command and does **not** pick up a changed Custom Start Command until a genuinely fresh deploy is triggered (in the CLI: `serviceInstanceDeploy`, not `deploymentRedeploy`; in the dashboard, this generally means triggering a new deploy from the service settings, not just clicking "Redeploy" on an existing deployment). If Redis auth fails right after editing the start command, trigger an actual new deployment, not a redeploy of the old one.
- **Uploads fail / 403 on file attach:** `AWS_S3_ENDPOINT_URL` wasn't set, or doesn't point at `minio`'s private domain. Without it, the app defaults to real AWS S3 and MinIO credentials fail auth there.
- **App won't start, domain validation error in logs:** `DOMAIN_NAME` has a `https://` prefix. It must be the bare hostname only.
- **Real-time collaboration (live cursors, live editing) doesn't work but everything else does:** `LIVE_SERVER_SECRET_KEY` mismatch or missing — this key must be set (the API and live server both read it independently; there's no cross-service reference to enforce this at the platform level).
- **This is a genuinely heavy template to run** — 5 services, comparable to Postiz's 8-service footprint. Not a cheap template; be upfront about this in the docs rather than downplaying it.

---

## 7. Post-Deploy Steps

After the template is published, test-deploy from a fresh Railway account (incognito window) and verify:

1. No "needs configuration" prompts appear for any variable.
2. All 5 services reach a healthy running state (Postgres migrations complete — allow 1-2 minutes on first boot).
3. The web UI loads at the generated domain and the account-creation / workspace-setup flow completes.
4. Create a test issue with a file attachment — confirms MinIO wiring (`AWS_S3_ENDPOINT_URL`, credentials) is genuinely correct, not just present.
5. Confirm the icon is set (1:1, transparent, Plane's real mark — not a generic placeholder).
