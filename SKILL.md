---
name: mailhub-p0-constraint
description: "Binding, non-negotiable implementation contract for the MailHub white-label email/SMS/MMS marketing platform (P0). Use when building, modifying, reviewing, deploying, or testing MailHub: any work in the mailhub/ project, any backend code (FastAPI / zeep SOAP adapter / arq workers), the React white-label frontend, or anything touching SpreadWS (rspread) integration, unsubscribe / compliance two-way sync, campaign queueing, idempotency, circuit breaker, or multi-tenant isolation. This skill is a CONSTRAINT, not a suggestion - the embedded spec is the sole authority and silent deviations are prohibited. When a user request appears to conflict with the spec, STOP and ask rather than improvising."
---

# MailHub P0 — Binding Implementation Contract

This skill is a **constraint layer**. The full authoritative specification lives in
`references/mailhub-implementation-spec.md` (a copy of the user's `mailhub-implementation-spec.md`).
Any MailHub work MUST be governed by that spec. The rules below are extracted from it and are
enforced as hard requirements. Treat every rule here as a "reject and rewrite" condition.

## Before any MailHub code change

1. Load `references/mailhub-implementation-spec.md` into context.
2. Confirm the change maps to an existing section of the spec (tech stack, structure, constraint,
   SQL, code example, implementation order, or acceptance criterion).
3. If the change is **not** in the spec, do NOT invent it. Surface the gap to the user and wait for
   explicit confirmation before proceeding. Never "improve" or "polish" beyond the spec.

## Tech-stack lock (do not change)

| Layer | Technology | Notes |
|---|---|---|
| Language | Python 3.11+ | zeep SOAP client |
| Web | FastAPI 0.110+ | async-native, auto OpenAPI |
| SOAP | zeep 4.2+ | only mature Python SOAP lib |
| DB | PostgreSQL 16+ | JSONB, row-level safety |
| Queue/Cache | Redis 7+ + arq (BullMQ-style) | NOT Celery |
| ORM | SQLAlchemy 2.0+ async | Alembic migrations |
| Validation | Pydantic 2.x | |
| HTTP | httpx 0.27+ | NOT requests |
| Cron | arq cron | sync engine / health check |
| Tests | pytest + pytest-asyncio 8+ | |
| Frontend | React 18 + Vite + Tailwind | white-label skinnable |
| Deploy | Docker Compose | one-command start |

**Forbidden to introduce:** Django, Flask, Celery, requests. Use arq (not Celery) and httpx (not requests).

## The 10 hard constraints (violating any = reject & rewrite)

1. **Spread credentials safety** — `SPREAD_LOGIN_EMAIL` / `SPREAD_PASSWORD` only from env vars.
   Never write to DB, logs, or the frontend. SOAP calls go to
   `https://service.rspread.com/Service.asmx` only. No `http://`.
2. **SOAP only via adapter** — business layer (`services/`) must NOT import zeep or build SOAP
   envelopes. All SOAP goes through `adapter/spread_driver.py`. Adapter returns a unified
   `ApiResponse`; raw SOAP is never exposed.
3. **Idempotency required** — every create endpoint (`createCampaign`, `addSubscriber*`) accepts an
   `idempotency_key`. Check `send_job` first; if the key exists return the prior result without
   calling SOAP. Key is client-generated (UUID); server only dedupes.
4. **Two-way unsubscribe sync** — Path A: local unsubscribe → mark local + call
   `UpdateSubscriberStatus` + write `sync_log`. Path B: Sync Engine polls Spread every 5 min,
   detects diffs, writes back locally + `sync_log`. Both paths are mandatory.
5. **Send via queue** — `createCampaign` does NOT call SOAP directly. Write `send_job` + enqueue to
   Redis. Send Worker consumes, calls the adapter, updates `send_job.status`. User gets `queued`
   immediately, no SOAP wait.
6. **Unified error mapping** — Spread `"Success"/"Fail"/"Error"/"1"/"-1"` all map to `ApiResponse`
   (code `0` success, `!=0` failure). Frontend never sees raw SOAP.
7. **Multi-tenant isolation** — every table has `tenant_id`. Every query filters `tenant_id`. API
   gateway injects `tenant_id`. No model may omit `tenant_id`.
8. **Upload safety** — max 10 MB; extension whitelist `.csv/.xls/.xlsx/.zip`. Store to object
   storage first, parse in an async worker (not inline).
9. **All external calls time out** — SOAP 30s, HTTP 15s, DB query 10s.
10. **Logging** — use `structlog`, JSON output. Mask sensitive fields (email, password, api_key).
    Log every SOAP call: method, duration, success/failure, error code.

## Forbidden items (writing any = reject)

- `import zeep` in the business layer
- `SPREAD_PASSWORD` anywhere in frontend code
- `createCampaign` calling SOAP directly (must queue)
- unsubscribe that writes local only without calling Spread (must be two-way)
- SOAP over `http://` (must be `https://`)
- logging email / password / api_key in plaintext
- creating `send_job` without the idempotency check
- using the `requests` library (use `httpx`)
- using Celery (use `arq`)
- any model missing `tenant_id`

## Implementation order (strict — follow exactly)

1. Infra: `config.py`, `database.py`, `models/base.py`, Alembic init + `001_initial.sql`
2. Adapter (core, write first): `error_mapper` → `spread_client` → `base` (ABC) → `circuit_breaker`
   → `retry` → `idempotency` → `spread_driver`
3. Data models: all 14 tables + Alembic migration
4. Services: `contact_service`, `list_service`, `campaign_service` (queue mode),
   `compliance_service` (unsubscribe loop)
5. API routes: `api/deps` (auth + tenant) → `auth`, `contacts`, `lists`, `campaigns`, `emails`,
   `unsubscribe`, `health`
6. Workers: `settings`, `send_worker`, `sync_engine`, `health_check`
7. Cross-cutting: `core/security` (JWT), `core/rate_limit`, `core/cache`
8. Frontend (P0 simplified): React+Vite+Tailwind init → Sidebar → Dashboard/Contacts/Lists/SendEmail
   → wire to backend API
9. Tests: `test_adapter`, `test_api`, `test_services`
10. Docker: `Dockerfile`, `docker-compose.yml`, `.env.example`

## White-label frontend fidelity rule (strict, no embellishment)

The frontend MUST be a 1:1 faithful port of the **approved design** (the prototype / design doc the
user has signed off, e.g. `spread-whitelabel-design.md` when present). Specifically:

- Use the exact design tokens from the approved doc (brand color, canvas color, ink, border, radius,
  sidebar width). Do NOT substitute your own palette or "nicer" values.
- Port the approved DOM/CSS **verbatim**. Do NOT add shadows, glows, icons, animations, extra spacing,
  or any element that is not in the approved design.
- Do NOT let a CSS framework's reset/preflight silently strip the approved styling. If Tailwind
  preflight conflicts with the approved look, disable preflight and paste the approved style block as-is.
- If a page needs a field/section not in the approved design, ask the user first — do not add it.
- "Make it prettier" is NOT a valid instruction when it contradicts the approved design. The
  approved design is the source of truth.

## Acceptance criteria (all must pass before declaring done)

- [ ] `docker compose up` starts api + worker + db + redis cleanly
- [ ] `POST /v1/auth/token` returns a JWT
- [ ] `POST /v1/contacts` adds a contact: local DB row + Spread side both present
- [ ] `POST /v1/contacts/{id}/status {"status":"Unsubscribed"}` marks local + Spread (two-way)
- [ ] `POST /v1/campaigns/email` returns `queued` (no direct send)
- [ ] Send Worker consumes the queue and sets `campaign.status = completed`
- [ ] Same `idempotency_key` repeated → returns existing result, no duplicate SOAP call
- [ ] Sync Engine runs every 5 min and writes `sync_log` rows
- [ ] Spread SOAP failing 5 times consecutively → Circuit Breaker opens, returns code `9100`
- [ ] All APIs return unified `ApiResponse`; frontend never sees raw SOAP

## Deviation handling (mandatory)

If a user request, a discovered bug, or an environment limitation seems to require departing from this
spec (different library, different structure, skipping a constraint, changing the design), **STOP**.
Do not silently comply. State the conflict explicitly and ask the user to confirm the deviation. Only
after explicit confirmation may the spec be treated as amended for that one point. The default
behavior is strict adherence.

## References

- Full spec (authoritative): `references/mailhub-implementation-spec.md`
  — contains the complete project structure, all 14 table CREATE SQL, every code example, `.env.example`,
  and `docker-compose.yml`. Load it before writing MailHub code.
