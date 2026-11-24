---
name: vps-docker-deploy
description: Configures Docker Compose, app-level session cookie auth, Alembic migrations, SQLite WAL mode with safe backups, and nginx reverse-proxy routing for deploying the CRM alongside n8n on an Ubuntu VPS. Use whenever creating or modifying Dockerfiles, docker-compose.yml, nginx configs, startup scripts, or database connection config.
---
# VPS & Docker Deployment Standards

## 1. Docker Compose & Networking

- Package the CRM into a lean container (`python:3.12-slim`), target <150 MB RAM.
- Attach the CRM service to the existing external Docker network used by n8n (configurable via `N8N_DOCKER_NETWORK`, default `n8n_default`) so n8n and the CRM communicate over internal Docker DNS (`http://tutor-crm:8000`) without public internet round-trips.
- Mount `./data:/app/data` as a persistent volume for the SQLite database (`crm.db`, `crm.db-wal`, `crm.db-shm`).
- The frontend is served by FastAPI (Jinja2 templates + static files) — no separate frontend container or build step.

## 2. Auth Architecture

**Browser UI — app-level session cookie auth:**
- FastAPI owns authentication. nginx is a plain reverse proxy with no auth logic.
- Login flow: `GET /login` → styled Jinja2 form → `POST /login` → check password → set signed cookie → redirect to `/`.
- Password check: compare submitted password against `APP_PASSWORD` env var using `secrets.compare_digest` (timing-safe). No user table needed for a single-user app.
- Session cookie: signed with `APP_SECRET_KEY` using `itsdangerous.TimestampSigner`. Store only `{"authenticated": true}` in the cookie payload. Set `httponly=True`, `samesite="lax"`, `secure=True` in production.
- Auth middleware: a Starlette `BaseHTTPMiddleware` that checks the signed cookie on every request. Redirects unsigned/expired requests to `/login`. Exempt paths: `/login`, `/static/`, `/api/webhooks/`.
- Session lifetime: configurable via `SESSION_MAX_AGE_SECONDS` (default 86400 × 30 — 30 days).

**n8n Webhooks — secret header:**
- `/api/webhooks/` routes are exempt from the session middleware (n8n cannot hold a cookie).
- FastAPI middleware validates `X-Webhook-Secret` header against `WEBHOOK_SECRET` env var on all `/api/webhooks/` routes. Return 401 on mismatch.

## 3. Database Portability & Startup Migrations

- Configure the database exclusively via `DATABASE_URL` in `.env` (default: `sqlite+aiosqlite:///./data/crm.db`).
- On SQLAlchemy engine connect (SQLite dialect only), execute:
```sql
  PRAGMA journal_mode=WAL;
  PRAGMA synchronous=NORMAL;
  PRAGMA foreign_keys=ON;
  PRAGMA busy_timeout=5000;
```
- Container entrypoint must run `alembic upgrade head` before starting `uvicorn`. Never call `Base.metadata.create_all()` in production.

## 4. Safe SQLite Backups

- Use `sqlite3 .backup` (not `cp`) — safe while WAL mode is active:
```bash
  sqlite3 /app/data/crm.db ".backup '/app/data/backups/crm_$(date +%F_%H%M).db'"
```
- Prune backups older than 14 days after each run.
- Schedule via a cron job on the VPS or a cron container in Compose.
- Provide this logic in `scripts/backup_db.sh`.

## 5. Environment Variables (`.env.example`)

Always keep `.env.example` current:
```
DATABASE_URL=sqlite+aiosqlite:///./data/crm.db
APP_PASSWORD=
APP_SECRET_KEY=
SESSION_MAX_AGE_SECONDS=2592000
WEBHOOK_SECRET=
N8N_DOCKER_NETWORK=n8n_default
N8N_WEBHOOK_URL_LEAD_CREATED=
ALLOWED_ORIGINS=https://crm.yourdomain.com
```
