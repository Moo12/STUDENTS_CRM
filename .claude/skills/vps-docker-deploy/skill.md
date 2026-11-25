---
name: vps-docker-deploy
description: Configures Docker Compose, app-level session cookie auth, Alembic migrations, SQLite WAL mode with safe backups, and nginx reverse-proxy routing for deploying the CRM alongside n8n on an Ubuntu VPS. Use whenever creating or modifying Dockerfiles, docker-compose.yml, nginx configs, startup scripts, or database connection config.
---
# VPS & Docker Deployment Standards

## 1. Docker Compose & Networking

The full stack on the VPS consists of four containers, all on the `agency-net` Docker network:

| Container | Image | Role | RAM target |
| --- | --- | --- | --- |
| `n8n-welder` | `n8nio/n8n:latest` | Automation (pre-existing) | ~372 MB |
| `tutor-crm` | (this project) | FastAPI CRM app | <150 MB |
| `evolution-api` | `atendai/evolution-api` | WhatsApp ↔ webhook bridge | ~300 MB |
| `nginx` | `nginx:alpine` | Reverse proxy | ~50 MB |

**Networking rules:**
- All containers join the existing external network `agency-net` — declare it as `external: true` in `docker-compose.yml`, never recreate it
- Internal addresses: `http://n8n-welder:5678`, `http://tutor-crm:8000`, `http://evolution-api:8080`
- Only nginx exposes ports to the host (80/443)
- Mount `./data:/app/data` for the CRM SQLite database (`crm.db`, `crm.db-wal`, `crm.db-shm`)
- The frontend is served by FastAPI (Jinja2 templates + static files) — no separate frontend container or build step

## 1b. Evolution API — WhatsApp Bridge

Evolution API connects to WhatsApp Web via QR scan and forwards incoming messages as webhooks to n8n.

**Setup steps (one-time):**
1. Start the `evolution-api` container
2. Create an instance via API: `POST http://localhost:8080/instance/create`
3. Fetch the QR code: `GET http://localhost:8080/instance/connect/{instanceName}`
4. Scan with your WhatsApp — connection persists across restarts via the mounted volume
5. Register n8n's webhook URL in Evolution API so it fires on every incoming message

**docker-compose snippet:**
```yaml
evolution-api:
  image: atendai/evolution-api:latest
  container_name: evolution-api
  restart: unless-stopped
  volumes:
    - ./evolution-data:/evolution/instances
  environment:
    - AUTHENTICATION_API_KEY=${EVOLUTION_API_KEY}
    - WEBHOOK_GLOBAL_URL=${N8N_EVOLUTION_WEBHOOK_URL}
    - WEBHOOK_GLOBAL_ENABLED=true
    - WEBHOOK_EVENTS_MESSAGES_UPSERT=true
  networks:
    - agency-net

networks:
  agency-net:
    external: true
```

**n8n receives** a POST from Evolution API when a message arrives. The n8n workflow filters by message content matching the agency template, then runs the parsing Code node and POSTs to the CRM ingest endpoint.

**Evolution API key** (`EVOLUTION_API_KEY`) is a secret you set — add it to `.env` and `.env.example`.
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
# Docker network shared with n8n (already exists on VPS as "agency-net")
N8N_DOCKER_NETWORK=agency-net
# n8n internal address within agency-net
N8N_BASE_URL=http://n8n-welder:5678
N8N_WEBHOOK_URL_LEAD_CREATED=
ALLOWED_ORIGINS=https://crm.yourdomain.com
```
