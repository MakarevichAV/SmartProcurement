# Development setup

Three repositories in one workspace (`SmartProcurement/`):

| Path | Repo | Stack |
|------|------|-------|
| `./` | `SmartProcurement` | Spec Kit artifacts + docs (this repo) |
| `./backend/` | `SmartProcurement-Backend` | Python 3.12, FastAPI, PostgreSQL |
| `./frontend/` | `SmartProcurement-Frontend` | React 18, TypeScript, Vite |

`backend/` and `frontend/` are standalone git repos, git-ignored by the root.

## Prerequisites

- Python 3.12 + [uv](https://docs.astral.sh/uv/)
- Node 20+ (developed against Node 24)
- Docker (for local PostgreSQL) — or an external PostgreSQL 16

## Local infrastructure

```sh
docker compose up -d postgres
```

| Service | Host port | Container | Credentials |
|---------|-----------|-----------|-------------|
| PostgreSQL 16 | **5432** | `smartprocurement-postgres` | user `sp` / pass `sp` / db `smart_procurement` |

DSN: `postgresql+asyncpg://sp:sp@localhost:5432/smart_procurement`.
`pgvector` is **not** used in v1.

## Backend

```sh
cd backend
uv sync --extra dev
cp .env.example .env            # set FERNET_KEY for anything beyond local
uv run uvicorn app.main:app --reload    # http://localhost:8000
# health:  http://localhost:8000/health
# OpenAPI: http://localhost:8000/openapi.json
uv run pytest
```

`make` targets: `install lint format typecheck test run worker migrate seed`.

## Frontend

```sh
cd frontend
npm install
cp .env.example .env            # VITE_API_BASE_URL, default http://localhost:8000
npm run dev                     # http://localhost:5173
npm run build                   # type-check + production bundle
npm run lint
npm run gen:api                 # regenerate src/api/schema.d.ts from the running backend
```

## Port summary

| Port | Service |
|------|---------|
| 5432 | PostgreSQL |
| 8000 | Backend API (uvicorn) |
| 5173 | Frontend dev server (Vite) |
