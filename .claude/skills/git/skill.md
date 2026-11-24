---
name: git
description: Git workflow, commit conventions, and branch strategy for the tutoring CRM. Use whenever committing, branching, pushing, or reviewing git history in this project.
---

# Git Practices — Tutoring CRM

## Repository

- **Remote:** `git@github.com:Moo12/STUDENTS_CRM.git`
- **Default branch:** `main`
- **Visibility:** private

## What Never Gets Committed

- `.env` — contains secrets (`APP_PASSWORD`, `APP_SECRET_KEY`, `WEBHOOK_SECRET`, `GH_TOKEN`)
- `data/` — SQLite database and backups (live data, potentially sensitive)
- Any file matching `.env.*` except `.env.example`

If you accidentally stage one of these, unstage immediately: `git reset HEAD <file>`.

## Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Always deployable. Direct commits only for single-line fixes or doc changes. |
| `feat/<slug>` | New features (e.g., `feat/lead-ingest-webhook`) |
| `fix/<slug>` | Bug fixes (e.g., `fix/phone-normalization`) |
| `chore/<slug>` | Config, deps, migrations, tooling (e.g., `chore/alembic-init`) |

Merge to `main` via fast-forward when the branch is clean, or squash if the branch has noisy WIP commits.

## Commit Message Format

```
<type>(<scope>): <short summary in present tense>

<optional body — the WHY, not the WHAT>
```

**Types:** `feat`, `fix`, `chore`, `refactor`, `docs`, `test`

**Scopes:** `auth`, `webhooks`, `leads`, `ui`, `db`, `deploy`, `skills`

**Rules:**
- Summary line ≤ 72 characters, no period at end
- Use present tense: "add phone normalization" not "added"
- Body explains *why* a change was made, not what — the diff shows the what
- Reference the relevant skill if the change follows one (e.g., "follows n8n-webhook-contract")

**Examples:**
```
feat(webhooks): add ingest endpoint for agency_whatsapp source

fix(auth): use secrets.compare_digest to prevent timing attacks

chore(db): run alembic init and create leads migration

docs(skills): update vps-docker-deploy with nginx config details
```

## Before Every Push

1. Confirm `.env` is not staged: `git status`
2. Confirm `data/` is not staged
3. Run a quick sanity check if touching backend: `git diff main..HEAD --stat`

## Alembic Migrations

Always commit migration files (`alembic/versions/*.py`) together with the model change that triggered them, in the same commit. Never commit a migration without the corresponding model, or vice versa.

## `.env.example`

Keep `.env.example` committed and up to date. Every new env var added to the app must appear in `.env.example` with an empty or placeholder value and a one-line comment explaining it.
