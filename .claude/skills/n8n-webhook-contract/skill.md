---
name: n8n-webhook-contract
description: Enforces standardized JSON envelope payloads, Israeli/international E.164 phone normalization, ORM-portable idempotent upserts, and WEBHOOK_SECRET-signed event triggers between the tutoring CRM and self-hosted n8n. Use whenever building or editing webhook endpoints, ORM models for leads/triggers, or n8n workflow payloads.
---

# n8n Webhook & ORM Data Contract

## 1. Standard Payload Envelope

All payloads — inbound (n8n → CRM) and outbound (CRM → n8n) — use this envelope:

```json
{ "source": "<source_slug>", "event": "<event_name>", "payload": { ... } }
```

- `source`: stable slug identifying the lead origin (e.g., `agency_whatsapp`, `website_form`)
- `event`: action name (e.g., `new_lead`, `lead_updated`, `follow_up_due`)
- `payload`: event-specific fields; source-specific extras go here and are stored in `Student.custom_metadata` (`sqlalchemy.JSON`) — never as hardcoded columns

## 2. Incoming Lead Ingestion (`POST /api/webhooks/ingest/{source_slug}`)

- **Authentication:** Require `X-Webhook-Secret` header validated against `WEBHOOK_SECRET` env var in FastAPI middleware. Return 401 on mismatch.
- **Phone Normalization:** Normalize all incoming phone numbers to strict E.164 before querying or saving:
  - Strip spaces, dashes, parentheses, leading `+` or `00`
  - Israeli local numbers starting with `0` (e.g., `054-1234567`) → `+972541234567`
  - Numbers starting with `972` → `+972...`
- **ORM-Portable Idempotency:** Never use SQLite-specific `INSERT OR IGNORE` or raw `ON CONFLICT`. Implement deduplication in the service layer via standard SQLAlchemy `SELECT` + `UPDATE`/`INSERT` inside an async transaction.
  - Deduplicate `Parent` by normalized `phone_e164`
  - Deduplicate `Student` by `(parent_id, normalized_student_name, source_id)` — allows siblings, prevents duplicate bot alerts

### Ingestion Request Body (inside envelope `payload`)

```json
{
  "external_lead_id": "optional-message-or-row-id",
  "parent": {
    "name": "David Cohen",
    "phone": "054-1234567",
    "email": null
  },
  "student": {
    "name": "Noa Cohen",
    "grade": "11th Grade - 5 Units Math",
    "notes": "Needs help before winter Bagrut",
    "custom_metadata": {
      "raw_whatsapp_text": "..."
    }
  }
}
```

Full request:
```json
{
  "source": "agency_whatsapp",
  "event": "new_lead",
  "payload": { ... }
}
```

## 3. Outbound Events (CRM → n8n)

- POST to n8n webhook URLs stored in env vars (e.g., `N8N_WEBHOOK_URL_LEAD_CREATED`)
- Include `X-Webhook-Secret` header with `WEBHOOK_SECRET` value
- Use the same envelope format; n8n validates the secret on its end

## 4. Response Contract

Ingest endpoint must return:
```json
{ "status": "created" | "updated", "student_id": "<uuid>", "parent_id": "<uuid>" }
```
