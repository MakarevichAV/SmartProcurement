# Development setup

Three repositories in one workspace (`SmartProcurement/`):

| Path | Repo | Stack |
|------|------|-------|
| `./` | `SmartProcurement` | Spec Kit artifacts + docs (this repo) |
| `./backend/` | `SmartProcurement-Backend` | Python 3.12, FastAPI, PostgreSQL |
| `./frontend/` | `SmartProcurement-Frontend` | React 18, TypeScript, Vite |

`backend/` and `frontend/` are standalone git repos, git-ignored by the root.

## Branching

Each of the three repos uses the same convention:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feature/<slug>` | new functionality / a spec phase | `feature/foundational` |
| `fix/<slug>` | bug fixes (introduced as needed) | `fix/policy-expiry-scan` |
| `main` | integration branch; always runnable | — |

A change that spans repos (e.g. a REST contract update touching backend and frontend) uses
the **same branch name in every affected repo**, so the coordinated work is easy to find and
review together. Branch off `main`; merge back to `main` when the slice is done.

## Status

Phase 1 (Setup) and Phase 2 (Foundational) are complete, including a UI/design foundation for
the Phase 2 shell; Phase 3 has not started. The stack below is fully runnable: FastAPI +
PostgreSQL, auth/refresh/RBAC, the capability registry and one-level promotion, append-only
audit, the job-queue worker, and the React app — a redesigned login, the responsive
authenticated navigation shell (design tokens + in-repo UI primitives) and a dashboard shell;
other business screens are placeholders. See the root `README.md` for the
implemented-vs-planned breakdown.

## Prerequisites

- Python 3.12 + [uv](https://docs.astral.sh/uv/)
- Node 20+ (developed against Node 24)
- Docker (for local PostgreSQL) — or an external PostgreSQL 16

## Local infrastructure

```sh
docker compose up -d postgres
```

| Service | Host binding | Container | Credentials (defaults) |
|---------|--------------|-----------|------------------------|
| PostgreSQL 16 | **127.0.0.1:5432** (loopback only) | `smartprocurement-postgres` | user `sp` / pass `sp` / db `smart_procurement` |

DSN: `postgresql+asyncpg://sp:sp@localhost:5432/smart_procurement`.
`pgvector` is **not** used in v1.

Credentials and port are overridable via the environment (`POSTGRES_USER`, `POSTGRES_PASSWORD`,
`POSTGRES_DB`, `POSTGRES_PORT`) or a root `.env`. The defaults are throwaway local values; this
compose file is for local development only and binds Postgres to loopback so it is not
reachable from other machines.

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
