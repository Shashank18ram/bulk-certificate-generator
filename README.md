# Bulk Certificate Generator

A backend API that accepts **one request containing many recipients**, generates a PDF certificate for each
from a single predefined template, tracks progress per job, and lets the client download the results
(individually or as one ZIP).

**Stack:** Python 3.10+ · FastAPI · SQLAlchemy 2 (SQLite by default, any SQL DB via a URL) · Pydantic v2 · ReportLab · pytest

![sample](docs/sample-certificate.png)

---

## 1. Setup

```bash
git clone <https://github.com/Shashank18ram/bulk-certificate-generator/blob/main/README.md> bulk-certificate-generator
cd bulk-certificate-generator

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
```

## 2. Run the application

```bash
uvicorn app.main:create_app --factory --reload
```

* API: http://127.0.0.1:8000
* Interactive docs (Swagger): http://127.0.0.1:8000/docs
* The SQLite DB and generated PDFs are created under `./data/` on first start.

Optional configuration (environment variables, see `.env.example`):

| Variable | Default | Meaning |
|---|---|---|
| `CERT_DATABASE_URL` | `sqlite:///./data/certgen.db` | Any SQLAlchemy URL, e.g. `postgresql+psycopg://user:pw@host/db` |
| `CERT_STORAGE_DIR` | `./data/certificates` | Where PDFs are written |
| `CERT_MAX_RECIPIENTS` | `1000` | Max recipients per request (413 above this) |
| `CERT_WORKER_MODE` | `thread` | `thread` (background pool), `inline` (synchronous), `manual` (tests) |
| `CERT_MAX_WORKERS` | `2` | Concurrent jobs processed |
| `CERT_FONT_PATH` | – | Path to a Unicode `.ttf` to support non-Latin names |

## 3. Run the tests

```bash
pytest
```

36 tests; they use a temporary SQLite DB and temporary storage per test, no setup required.

## 4. Submit a certificate generation request

`POST /api/v1/jobs`

```bash
curl -X POST http://127.0.0.1:8000/api/v1/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "event_name": "Python Bootcamp 2026",
    "issued_by": "Acme Academy",
    "issue_date": "2026-03-15",
    "recipients": [
      {"name": "Asha Rao",  "email": "asha@example.com"},
      {"name": "Ravi Kumar","email": "ravi@example.com", "award": "Distinction"},
      {"name": "No Email"}
    ]
  }'
```

| Field | Required | Notes |
|---|---|---|
| `event_name` | yes | 1–120 chars |
| `issued_by` | no | default "The Organizing Committee" |
| `issue_date` | no | ISO date, default today |
| `recipients[]` | yes | at least 1, at most `CERT_MAX_RECIPIENTS` |
| `recipients[].name` | yes | 1–100 chars, no control characters |
| `recipients[].email` | yes | valid email, unique within the request (case-insensitive) |
| `recipients[].award` | no | printed as "with *award*" (≤100 chars) |

Response `202 Accepted` (+ `Location` header):

```json
{
  "id": "6f1c…",
  "status": "PENDING",
  "total": 3, "succeeded": 0, "failed": 1, "in_progress": 2,
  "progress_percent": 33.3,
  "failures": [
    {"position": 3, "recipient_name": "No Email", "status": "FAILED",
     "error": "Invalid recipient: email: Field required"}
  ],
  "links": {"self": "/api/v1/jobs/6f1c…", "certificates": "/api/v1/jobs/6f1c…/certificates", "download_all": "/api/v1/jobs/6f1c…/download"}
}
```

### Check progress

`GET /api/v1/jobs/{job_id}` – poll this.

| Job `status` | Meaning |
|---|---|
| `PENDING` | accepted, waiting for a worker |
| `PROCESSING` | being generated (`progress_percent` / `succeeded` / `failed` update live) |
| `COMPLETED` | all certificates succeeded |
| `COMPLETED_WITH_ERRORS` | some succeeded, some failed – see `failures` |
| `FAILED` | nothing succeeded (or an unexpected job-level error, see `error_message`) |

`failures` lists every failed recipient with its `position` in the request, and a human-readable `error`.

## 5. Retrieve generated certificates

| Endpoint | Purpose |
|---|---|
| `GET /api/v1/jobs/{job_id}/certificates?status=SUCCEEDED&limit=100&offset=0` | Paginated per-recipient list, each success has a `download_url` |
| `GET /api/v1/certificates/{certificate_id}/download` | One PDF |
| `GET /api/v1/jobs/{job_id}/download` | ZIP of all successful PDFs (`409` while the job is still running) |

```bash
curl -o all.zip http://127.0.0.1:8000/api/v1/jobs/<job_id>/download
curl -OJ http://127.0.0.1:8000/api/v1/certificates/<certificate_id>/download
```

---

## 6. Design decisions

### Processing model: asynchronous, in-process background workers
`POST /jobs` validates, **stores everything, returns `202` immediately**, and a background thread pool
generates the PDFs. The client polls `GET /jobs/{id}`.

*Why not synchronous?* 1000 certificates in one HTTP request means a long-held connection, proxy/client
timeouts, and a lost result if the connection drops. *Why not Celery/Redis?* For this scope it adds
infrastructure (broker, worker process) without changing the API contract. Everything goes through a tiny
runner interface (`app/worker.py`: `submit(job_id)`), so swapping in Celery/RQ/ARQ later means changing one
class – the DB is already the source of truth for state.

### Data model (2 tables)
* `jobs` – request-level data and status.
* `certificates` – **one row per recipient submitted**, including invalid ones (stored as `FAILED` with the
  reason and the raw input). This gives the client a complete per-recipient report and keeps `position`
  stable so they can map results back to their input rows.

Progress counters are **computed from `certificates` with a `GROUP BY`** rather than stored on the job, so
they can never drift out of sync with reality.

### Validation: two levels
1. **Request level** (`JobCreate`): missing `event_name`, empty/non-list `recipients`, bad date → whole request
   rejected with `422`. Too many recipients → `413`.
2. **Recipient level** (`RecipientInput`): a bad email/name only fails *that recipient*. Recipients are
   deliberately typed as `list[Any]` in the request schema and validated one by one, otherwise Pydantic would
   reject the entire batch for one typo. Duplicate emails within a request are also flagged.

### Failure isolation
Each certificate runs in its own `try/except` and its own DB commit. A failure is stored on that row
(`FAILED` + error), everything else continues. The job ends as `COMPLETED`, `COMPLETED_WITH_ERRORS` or
`FAILED`. Per-certificate commits also mean progress is visible live.

### Reliability
* **Atomic claim:** `UPDATE jobs SET status='PROCESSING' WHERE id=? AND status='PENDING'` – if two workers
  race for the same job, only one wins.
* **Crash recovery:** on startup, jobs stuck in `PROCESSING` are reset to `PENDING` and re-submitted.
  The worker only picks up `PENDING` certificates, so already-finished ones are not regenerated.
* **Atomic file writes:** PDFs are written to `*.tmp` then renamed, so a crash never leaves a corrupt PDF.

### Certificate template
One hard-coded landscape A4 template (`app/generator.py`, ReportLab): name, event, optional award, date,
issuer and a unique certificate ID. Long names auto-shrink to fit; text that still doesn't fit fails
*that* certificate with a clear message.

### Storage
PDFs live on local disk at `storage/<job_id>/<certificate_id>.pdf`; the DB stores the *relative* path, so the
storage folder can be moved. Behind a storage interface this could become S3.

---

## 7. Known limitations / what I'd do next
* **Non-Latin names** (e.g. Devanagari, Chinese): the built-in PDF fonts only cover Latin-1, so such
  certificates fail with a clear error instead of rendering garbage. Set `CERT_FONT_PATH` to a Unicode TTF
  (e.g. Noto Sans) to support them. (Complex scripts like Devanagari would additionally need text shaping.)
* In-process thread pool: jobs only run while the API process is up (recovery handles restarts). For
  horizontal scaling, move to a real queue (Celery/RQ) + Postgres + S3.
* No authentication / rate limiting / idempotency keys – add API keys and an `Idempotency-Key` header
  to make retried `POST`s safe.
* Schema is created with `create_all`; use Alembic migrations in production.
* Possible extras: webhook/callback on completion, emailing certificates, retrying only failed recipients.

## 8. Project layout

```
app/
  main.py        app factory, wiring, startup recovery
  api.py         HTTP endpoints
  service.py     job creation, validation, generation loop, recovery
  generator.py   PDF template
  worker.py      thread / inline / manual runners
  models.py      SQLAlchemy models + status enums
  schemas.py     Pydantic request/response models
  config.py      env-based settings
  database.py    engine/session setup
tests/           36 tests (creation, validation, generation, status, failures, retrieval)
```
