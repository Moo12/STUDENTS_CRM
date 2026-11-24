# Tutoring CRM — Project Architecture & Rules

## Overview
A lightweight, self-hosted, single-user CRM for a private math and physics tutoring business, deployed via Docker Compose on an Ubuntu VPS alongside a self-hosted **n8n** instance.

## Core Architecture Principles
1. **Decoupled Automation:** The CRM handles data persistence, UI, and outgoing event emission. **n8n** handles all external channel parsing (WhatsApp bot notifications, future agency emails/webhooks, website forms) and downstream actions (WhatsApp welcome messages, calendar invites, follow-ups).
2. **Database-Agnostic ORM + Migrations (SQLite First, Postgres-Ready):**
  - Use **FastAPI + SQLAlchemy 2.0 (async) + Alembic** (or Node + Drizzle/Prisma if chosen) with `DATABASE_URL` configured via environment variable (`sqlite+aiosqlite:///./data/crm.db` by default).
  - Enable SQLite `PRAGMA journal_mode=WAL;`, `PRAGMA foreign_keys=ON;`, and `PRAGMA busy_timeout=5000;` on connection startup when running on SQLite.
  - **Portability Rule:** Never write raw SQLite-only SQL (`sqlite_master`, `INSERT OR IGNORE`, `json_extract` raw strings). Use ORM cross-dialect types (`sqlalchemy.JSON`, `DateTime(timezone=True)`, `String`, `Numeric(10, 2)`) and standard ORM upserts/queries so switching `DATABASE_URL` to PostgreSQL requires zero application rewrites.
  - Every schema change must be generated and applied via **Alembic migrations** (`alembic revision --autogenerate`), never `Base.metadata.create_all()` in production.
3. **Configurable Multi-Source Model:** Every lead belongs to a `Source` (e.g., tutoring agency WhatsApp bot, future agencies, direct website). Source-specific fields go into a portable `JSON` metadata column rather than hardcoded columns.
4. **Mobile-First UI:** Keep the web interface clean, fast, and responsive on mobile viewports for checking leads between lessons.
  - Stack: **HTMX + Alpine.js + Tailwind CSS**, rendered via Jinja2 templates inside the FastAPI app. No separate build step, no second container. Alpine handles light interactivity (modals, dropdowns, inline edits); HTMX handles partial page updates.
5. **Auth:**
  - Browser UI: **app-level session cookie auth**. Login form → `POST /login` → password checked against `APP_PASSWORD` env var via `secrets.compare_digest` → signed `itsdangerous` cookie → Starlette middleware guards all routes. nginx is a plain reverse proxy with no auth logic.
  - Session cookie: `httponly`, `samesite=lax`, `secure` in production. 30-day lifetime (`SESSION_MAX_AGE_SECONDS`), signed with `APP_SECRET_KEY`.
  - n8n → CRM inbound webhooks: exempt from session middleware; validated by `X-Webhook-Secret` header against `WEBHOOK_SECRET` env var.
  - CRM → n8n outbound events: POST to n8n webhook URLs stored in env vars, same `WEBHOOK_SECRET` in outgoing header.
5. **n8n Webhook Envelope:** All payloads (inbound and outbound) use a standard envelope:
```json
   { "source": "<source_slug>", "event": "<event_name>", "payload": { ... } }
```
   Source-specific fields go in `payload` and are stored in the ORM `JSON` metadata column — no hardcoded per-source columns.
7. **Backups:** Nightly `sqlite3 data/crm.db ".backup 'data/backups/crm_$(date +%F_%H%M).db'"` via a cron job on the VPS. `sqlite3 .backup` is WAL-safe (unlike `cp`). Keep last 14 days.
## Directory Structure
- `backend/` — FastAPI app, SQLAlchemy models, Alembic migrations, routers, webhook dispatcher
- `frontend/` — Server-rendered UI (HTMX + Alpine.js + Tailwind CSS via Jinja2 templates, served by FastAPI)
- `data/` — Mounted Docker volume for SQLite database and automated backups (gitignored)
- `.claude/skills/` — Project-specific Claude Code skills (`n8n-webhook-contract`, `vps-docker-deploy`)

## Mandatory Skill Usage
- When creating or modifying API endpoints, database models, or n8n payloads, follow `.claude/skills/n8n-webhook-contract/SKILL.md`.
- When modifying `Dockerfile`, `docker-compose.yml`, Alembic startup commands, Nginx configs, or backup scripts, follow `.claude/skills/vps-docker-deploy/SKILL.md`.