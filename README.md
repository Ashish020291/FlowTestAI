# FlowTestAI

AI-powered test automation platform. Write test cases in natural language, execute them
with Playwright across multiple browsers, and get intelligent failure analysis —
**no database required**.

#FlowTestAI

This project archive is split into 11 parts because of GitHub file-size limitations.

### Download

Download **all 11 parts**:

`FlowTestAI.partaa` → `FlowTestAI.partaak`

Keep all files in the **same folder**.

### Combine the files

**Windows (Command Prompt):**

```cmd
copy /b MyProject.zip.a+MyProject.zip.b+MyProject.zip.c+MyProject.zip.d+MyProject.zip.e+MyProject.zip.f+MyProject.zip.g+MyProject.zip.h+MyProject.zip.i+MyProject.zip.j+MyProject.zip.k MyProject.zip
```

**macOS / Linux:**

```bash
cat MyProject.zip.a MyProject.zip.b MyProject.zip.c MyProject.zip.d MyProject.zip.e MyProject.zip.f MyProject.zip.g MyProject.zip.h MyProject.zip.i MyProject.zip.j MyProject.zip.k > MyProject.zip
```

After combining, extract `MyProject.zip` normally.

**Important:** All 11 parts are required. Do not rename, modify, or extract the individual parts.


---

## Architecture

FlowTestAI stores all data in plain files under a single `data/` directory.
There is no PostgreSQL dependency; Redis is retained only for the job queue and SSE
pub/sub; S3 (optional) stores screenshots and video recordings.

```
data/
  users.yaml               # User accounts (bcrypt-hashed passwords)
  config.yaml              # Org settings, members, notifications, Jira config, API keys
  environments.yaml        # Environment definitions and variables
  credentials.yaml         # AES-256-GCM encrypted credentials (base64-encoded fields)
  schedules.yaml           # Cron schedules
  runs_index.json          # Flat index of run metadata for fast report aggregation
  audit.log                # Append-only newline-delimited JSON audit events

  suites/
    {suite_id}/
      meta.json            # Suite metadata
      suite.xlsx           # TestCases sheet + Steps sheet
      runs/{run_id}.json   # Full run + per-case + per-step results (atomic write)
```

All file writes go to a `.tmp` file first, then renamed atomically.
Per-file `asyncio.Lock` + `portalocker` guards concurrent access in multi-worker
deployments.

---

## Prerequisites

- Python 3.12+
- Node.js 20+ (frontend)
- Redis 7+
- AWS credentials (optional — screenshots and video storage)
- OpenAI-compatible API key (optional — AI test generation and analysis)

---

## Quick Start

### Docker Compose (recommended)

```bash
# 1. Copy env template and fill in the required values
cp .env.example .env

# 2. Start all services (no database setup needed)
docker compose up --build
```

The API is available at `http://localhost:8000`. Swagger UI at `/docs`.

### Running locally

```bash
# Backend
cd backend
pip install -r requirements.txt
DATA_DIR=data uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Worker (separate terminal)
cd worker
pip install -r requirements.txt
DATA_DIR=data python -m app.main

# Frontend (separate terminal)
cd frontend
npm install && npm run dev
```

---

## Configuration

All settings are read from environment variables (or a `.env` file).
Copy `.env.example` to `.env` and fill in the required values.

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATA_DIR` | Yes | `data` | Root directory for all file-based data |
| `SECRET_KEY` | Yes | — | JWT signing secret (minimum 64 characters) |
| `REDIS_URL` | Yes | `redis://localhost:6379/0` | Redis DSN |
| `ENCRYPTION_KEY` | Yes | — | Fernet key for credential encryption |
| `AWS_ACCESS_KEY_ID` | No | — | S3 credentials (screenshots and video) |
| `AWS_SECRET_ACCESS_KEY` | No | — | S3 credentials |
| `AWS_REGION` | No | `us-east-1` | S3 region |
| `S3_BUCKET` | No | — | S3 bucket name |
| `LLM_API_KEY` | No | — | OpenAI-compatible key for AI features |
| `LLM_MODEL` | No | `gpt-4o` | Model name for AI features |
| `SMTP_HOST` | No | `localhost` | SMTP server for email notifications |

Generate a Fernet key with:
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

---

## Export / Import

Test suites, cases, steps, environments, and credentials can be packaged into a
self-contained ZIP archive and imported into any FlowTestAI instance.

### ZIP archive layout

```
export.json               Manifest (version, org_id, export_date, counts)
suites/<uuid>.json        One file per suite (metadata only, no cases)
cases/<uuid>.json         One file per case including its ordered steps
environments/<uuid>.json  Environment records (when include_environments=true)
credentials/<uuid>.json   Encrypted credential records (when include_credentials=true)
```

### Credential portability

Credentials are exported with their **encrypted payloads** intact (the raw secret
is never serialised). They will only decrypt correctly on the destination instance
if it uses the **same `ENCRYPTION_KEY`** value. If the keys differ, the credentials
are imported successfully but will fail at runtime when the application tries to
decrypt them — update the `ENCRYPTION_KEY` on the destination, or re-create the
credentials manually.

### API

```
POST /api/v1/exports                    Create a ZIP export archive
GET  /api/v1/exports/{id}              Poll export status / get download URL
GET  /api/v1/exports/{id}/download     Download the completed ZIP
POST /api/v1/imports                   Upload and import a ZIP archive
```

#### Example

```bash
# Export all suites including environments and credentials
curl -X POST http://localhost:8000/api/v1/exports \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Organization-ID: $ORG_ID" \
  -H "Content-Type: application/json" \
  -d '{"include_environments": true, "include_credentials": true}'

# Download
curl "http://localhost:8000/api/v1/exports/$EXPORT_ID/download" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Organization-ID: $ORG_ID" \
  -o backup.zip

# Import on another instance
curl -X POST http://other-instance:8000/api/v1/imports \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Organization-ID: $ORG_ID" \
  -F "file=@backup.zip"
```

---

## CI/CD Integration (Public API)

Trigger test runs from pipelines using the `X-API-Key` header.
API keys are managed under **Settings → API Keys** in the dashboard.

```bash
# Trigger a run for a specific suite
curl -X POST http://localhost:8000/api/v1/public/trigger-run \
  -H "X-API-Key: ftai_<your_key>" \
  -H "Content-Type: application/json" \
  -d '{"suite_id": "<uuid>", "browser": "chromium"}'

# Poll for status
curl http://localhost:8000/api/v1/public/runs/<run_id> \
  -H "X-API-Key: ftai_<your_key>"

# Get full per-case report
curl http://localhost:8000/api/v1/public/runs/<run_id>/report \
  -H "X-API-Key: ftai_<your_key>"
```

Keys are stored SHA-256-hashed in `config.yaml` so that a file system breach
does not expose usable secrets.

---

## Worker

The background worker dequeues jobs from the `run_queue` Redis list, loads test
data from the suite's `suite.xlsx` Excel file, drives Playwright, and writes
results atomically to `suites/{suite_id}/runs/{run_id}.json`.

```bash
cd worker
DATA_DIR=data python -m app.main
```

---

## Jira Integration

Connect FlowTestAI to Jira to generate test cases from user stories and post
pass/fail comments back to issues.

1. Navigate to **Settings → Integrations → Jira** in the dashboard.
2. Enter your Jira base URL, project key, and a Personal Access Token (or API
   token for Jira Cloud).
3. Use **Generate Tests** on any test suite and paste a Jira issue key.

---

## Scheduled Runs

Create cron schedules under **Settings → Schedules**.
The APScheduler-based scheduler loads all active schedules at startup and
enqueues a run to Redis whenever a cron expression fires.

---

## Data Backup

Because all data lives in the `data/` directory, a full backup is:

```bash
tar -czf flowtestai-backup-$(date +%Y%m%d).tar.gz data/
```

Mount a persistent volume at the `DATA_DIR` path when running in Docker.
