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
    "name": "אלי",
    "phone": "972508214070",
    "email": null
  },
  "student": {
    "name": "דניאל דסה",
    "grade": "ח",
    "subject": "מתמטיקה",
    "address": "נחשון 6, תל אביב, תל אביב",
    "notes": null,
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

## 5. `agency_whatsapp` — n8n Parsing Logic

The tutoring agency bot sends a fixed-template Hebrew WhatsApp message. Parse it in n8n using a **Code node** (no LLM needed):

**Input message example:**
```
🎉 מעולה מעין, הבקשה אושרה! להלן הפרטים:

הנה הפרטים של דניאל דסה (כיתה ח) 😇

מקצוע: מתמטיקה
כתובת: נחשון 6, תל אביב, תל אביב
הורה: אלי | 972508214070

בבקשה ליצור קשר עם ההורה...
```

**n8n Code node (JavaScript):**
```js
const text = $input.first().json.message;

const studentMatch = text.match(/הנה הפרטים של (.+?) \(כיתה (.+?)\)/);
const subjectMatch = text.match(/מקצוע: (.+)/);
const addressMatch = text.match(/כתובת: (.+)/);
const parentMatch  = text.match(/הורה: (.+?) \| (\d+)/);

if (!studentMatch || !parentMatch) throw new Error('Message does not match agency_whatsapp template');

const rawPhone = parentMatch[2].trim();
const phone_e164 = '+' + rawPhone.replace(/^00/, '');

return [{
  json: {
    source: 'agency_whatsapp',
    event: 'new_lead',
    payload: {
      external_lead_id: null,
      parent: {
        name: parentMatch[1].trim(),
        phone: phone_e164,
        email: null,
      },
      student: {
        name: studentMatch[1].trim(),
        grade: studentMatch[2].trim(),
        subject: subjectMatch ? subjectMatch[1].trim() : null,
        address: addressMatch ? addressMatch[1].trim() : null,
        notes: null,
        custom_metadata: { raw_whatsapp_text: text },
      },
    },
  },
}];
```

**Grade mapping** (Hebrew letter → numeric, stored in `custom_metadata` if needed):
`א=1, ב=2, ג=3, ד=4, ה=5, ו=6, ז=7, ח=8, ט=9, י=10, יא=11, יב=12`

Store the raw Hebrew grade string in `student.grade` and add `student.grade_numeric` to `custom_metadata` if you need to filter/sort by grade numerically.
## 3. Outbound Events (CRM → n8n)

- POST to n8n webhook URLs stored in env vars (e.g., `N8N_WEBHOOK_URL_LEAD_CREATED`)
- Include `X-Webhook-Secret` header with `WEBHOOK_SECRET` value
- Use the same envelope format; n8n validates the secret on its end

## 4. Response Contract

Ingest endpoint must return:
```json
{ "status": "created" | "updated", "student_id": "<uuid>", "parent_id": "<uuid>" }
```
