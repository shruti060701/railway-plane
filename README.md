# Plane on Railway

Plane — open-source Jira/Linear alternative for project management, issue tracking, and cycles. Deploy on Railway with one click.

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/new/template/ZC7OPU)

## Architecture

This template deploys 5 services from the official `makeplane/plane-aio-community:v1.4.2` all-in-one image (bundles web, API, workers, and live-collaboration server behind a single Caddy proxy) plus its 3 required backing services:

- **plane** — the all-in-one Plane application (web UI, API, background workers, real-time collaboration)
- **postgres** — primary data store
- **redis** — caching and session store
- **rabbitmq** — background task queue (issue notifications, imports, exports)
- **minio** — S3-compatible object storage for file/attachment uploads

## How to use

Click the deploy button above, wait for all 5 services to finish deploying (Plane runs its own Postgres migrations on first boot, this can take 1-2 minutes), then open your generated Railway domain and create the first workspace admin account.

## Notes

- `DOMAIN_NAME` must exactly match your Railway-generated (or custom) domain, no protocol prefix. This is auto-wired by the template.
- Data persists across redeploys via 3 volumes: Postgres data, Redis AOF, and MinIO object storage.
- No custom Dockerfile — everything is pulled from official upstream images.

See `TEMPLATE_COMPOSER_CHECKLIST.md`, `TEMPLATE_DESCRIPTION.md`, and `TEMPLATE_BLOG.md` in this repo for the full build documentation.
