# yourqm_v1

Production-ready FastAPI backend and grounded AI agent API for **yourqm_v1 (Your Quality Manager)**, part of PCH Consulting — healthcare accreditation and compliance consulting for Australian providers (NSQHS, NSQPHC, RACGP, NDIS, Aged Care, cosmetics, digital mental health and more).

The service powers a website chat assistant that answers questions **only from approved, ingested yourqm_v1 content**, captures qualified leads, and exposes a small admin surface.

## Features

- **Grounded AI agent** — retrieval-augmented (BM25) answers constrained to ingested content, with a safe fallback when it has no evidence.
- **Provider-agnostic LLM layer** — OpenAI-compatible and Anthropic backends behind one interface.
- **Tool calling with safety rails** — an explicit tool registry, two-phase confirmation for sensitive tools (e.g. capturing a lead), and prompt-injection resistant handling.
- **Lead capture** — validated contact submissions, honeypot + rate limiting, optional forwarding to the existing Supabase `contact_submissions` table.
- **Sessions & feedback** — opaque session tokens, message history, and thumbs up/down feedback.
- **Admin API** — JWT-authenticated endpoints for leads, sessions and knowledge management.
- **Security by default** — Argon2id admin hashing, strict CORS, security headers, request body limits, structured redacted logging.

## Requirements

- Python 3.12+
- SQLite (local default) or PostgreSQL 14+ (production)
- Redis (optional — enables shared rate limiting across replicas)
- An LLM API key for AI responses (OpenAI-compatible or Anthropic)

## Quick start

```powershell
python -m venv .venv
& .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt

Copy-Item .env.example .env
# edit .env: set SECRET_KEY and AI_API_KEY at minimum

alembic upgrade head
python -m scripts.ingest_knowledge --source seed
uvicorn app.main:app --reload
```

Interactive docs: http://localhost:8000/docs · Health: http://localhost:8000/health

The API runs without an AI key; chat endpoints will return a clean `503 ai_not_configured` until `AI_API_KEY` is set.

## Configuration

All settings are environment variables (or `.env`). See [`.env.example`](.env.example) for the full list. Key values:

| Variable | Purpose |
| --- | --- |
| `SECRET_KEY` | Signing key for admin JWTs — required and must be strong in production. |
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/yourqm_v1.db` (default) or `postgresql+asyncpg://...`. |
| `REDIS_URL` | Enables Redis-backed rate limiting. |
| `AI_PROVIDER` / `AI_MODEL` / `AI_API_KEY` / `AI_BASE_URL` | LLM backend selection. |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD_HASH` | Admin login credentials. |
| `CORS_ORIGINS` | Comma-separated allowlist. |
| `KNOWLEDGE_URL_ALLOWLIST` | Hosts allowed for knowledge ingestion. |
| `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` | Optional lead forwarding to the existing site. |

Production startup refuses to boot when `SECRET_KEY` is left at its default or the admin password is unset.

### Admin password

```powershell
python -m scripts.generate_admin_hash --password "your-password"
# copy the printed ADMIN_PASSWORD_HASH into .env
```

## Knowledge ingestion

The agent answers only from ingested, approved content.

```powershell
# Bundled, verified seed content (recommended first run)
python -m scripts.ingest_knowledge --source seed

# Fetch public pages on allowlisted hosts
python -m scripts.ingest_knowledge --source urls
python -m scripts.ingest_knowledge --source urls --urls https://yourqm.com.au/nsfc
```

Only public pages on `KNOWLEDGE_URL_ALLOWLIST` hosts are fetched. Documents are re-chunked only when their content hash changes.

## Database migrations

```powershell
alembic revision --autogenerate -m "describe change"
alembic upgrade head
alembic check   # verify models and migrations agree
```

Development and tests use `Base.metadata.create_all`; production schema changes **must** go through Alembic.

## Testing & quality

```powershell
$env:APP_ENV="testing"
pytest
pytest --cov=app --cov-report=term-missing
ruff check .
ruff format .
```

The suite covers health, config, security primitives, retrieval, tools, orchestration, and the chat/leads/admin APIs (74 tests).

## API overview

Base path: `/api/v1` (health/readiness are also exposed at the root).

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Liveness probe (also `/api/v1/health`). |
| `GET` | `/ready` | Readiness probe — checks the database. |
| `POST` | `/api/v1/chat/sessions` | Create a chat session (returns a session token). |
| `POST` | `/api/v1/chat/sessions/{id}/messages` | Send a message; returns the reply. |
| `POST` | `/api/v1/chat/sessions/{id}/messages/stream` | Same, streamed as Server-Sent Events. |
| `GET` | `/api/v1/chat/sessions/{id}/messages` | Retrieve authorised conversation history. |
| `DELETE` | `/api/v1/chat/sessions/{id}` | Delete a session and its messages. |
| `POST` | `/api/v1/chat/feedback` | Submit feedback on an assistant response. |
| `POST` | `/api/v1/contact` | Submit a contact / enquiry request. |
| `POST` | `/api/v1/leads` | Capture a visitor enquiry. |
| `POST` | `/api/v1/admin/token` | Exchange credentials for a JWT. |
| `GET` | `/api/v1/admin/leads` | List captured leads. |
| `GET` | `/api/v1/admin/stats` | Operational statistics. |

Session ownership is enforced via the `X-Session-Token` header. Admin endpoints require `Authorization: Bearer <jwt>`.

## Deployment

### Docker Compose (PostgreSQL + Redis)

```powershell
Copy-Item .env.example .env   # set SECRET_KEY, ADMIN_PASSWORD_HASH, AI_API_KEY
docker compose up --build
```

`compose.yaml` provisions PostgreSQL 16 and Redis 7, and runs `alembic upgrade head` before starting Uvicorn.

### Standalone image

```powershell
docker build -t yourqm_v1 .
docker run --env-file .env -p 8000:8000 yourqm_v1
```

The container runs as a non-root user and exposes a `/health` healthcheck.

## Project layout

```
app/
  agents/         LLM providers, prompts, retrieval (BM25), tools, orchestration
  api/v1/         versioned routers and endpoint modules
  core/           config, logging, security, rate limiting, exceptions, dependencies
  db/             engine/session and declarative base
  middleware/     request context, security headers, body-size guard
  models/         SQLAlchemy ORM entities
  repositories/   data-access layer
  schemas/        Pydantic request/response models
  services/       business logic and Supabase bridge
alembic/          migration environment and versions
scripts/          knowledge ingestion and admin-hash CLIs
data/knowledge/   verified seed content
tests/            pytest suite
```

## License

Proprietary — © PCH Consulting / yourqm_v1. All rights reserved.
