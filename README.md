# Smart Procurement

An intelligent decision layer for enterprise procurement. The system connects to corporate
data sources, builds a single picture of the procurement situation, detects and explains risks
(shortages, missed lead times, price and quality anomalies), produces buying recommendations,
and — depending on the authority level under the **LORM** model — shows the decision to a
human, asks for confirmation, or executes the action autonomously within an approved policy.

**Goal**: give a mid-size manufacturer a trustworthy, auditable assistant for *what / how much
/ when / from whom* to buy — with authority that is granted deliberately, one step at a time,
and revoked automatically when something goes wrong.

Smart Procurement **does not replace** ERP/WMS/MES. It is a decision layer on top of existing
systems; execution and bookkeeping stay with them.

## Role of LORM (L0–L5)

The [Layered Operational Responsibility Model](https://github.com/Argyronix/lorm) assigns an
**authority level to each capability individually**, not to the system as a whole:

| Level | Meaning |
|-------|---------|
| **L0–L1** | Knows the domain / observes current data. |
| **L2** | Diagnoses a situation and explains it, with a confidence estimate. |
| **L3** | Recommends a concrete action; the human decides. |
| **L4** | Prepares an action and executes it **only after per-action human approval**. |
| **L5** | Executes autonomously **within the bounds of an approved machine-readable policy** (distinct author and approver). |

LORM governs *authority*, not the procurement decision itself. Level **promotion is human-only
and one step at a time**; **demotion is automatic** on policy violation, verification failure,
loss of observability, or policy expiry. Two capabilities (`proc.supplier.add`,
`proc.payment.release`) can never reach L5 — a system must not widen its own authority. The
official LORM policy schema and validator are vendored unmodified; enforcement is adapted to
the FastAPI runtime.

## Architecture

Layers (each communicates only through explicit interfaces; the backend is the only security
boundary):

```
Frontend → Backend/API → AI Decision Layer → LORM Responsibility/Enforcement
                       → Data/Integration → Execution → Persistence
```

The backend is a **modular monolith** (`src/app/<module>/`) plus a **worker process** sharing
the same code and database. The frontend is a single-page app that talks to the backend only
through a REST/OpenAPI contract and holds no trusted authorization logic.

## Technology stack

| Area | Stack |
|------|-------|
| Backend | Python 3.12 · FastAPI · Pydantic v2 · SQLAlchemy 2.0 (async) · Alembic · PostgreSQL 16 · `uv` |
| AI | `LLMProvider` seam — Anthropic SDK (real) / deterministic mock (tests); typed/validated output |
| Frontend | React 18 · TypeScript · Vite · Redux Toolkit + RTK Query · Tailwind CSS |
| Jobs | Durable Postgres-backed queue (`SELECT … FOR UPDATE SKIP LOCKED`), no external broker |
| LORM | Vendored `lorm-policy.schema.json` + `validate_policy.py`, pinned to an upstream commit |

## Repositories

Three separate Git repositories in one local workspace:

| Repository | Contents |
|------------|----------|
| [**SmartProcurement**](https://github.com/MakarevichAV/SmartProcurement) (this one) | Constitution, specifications, plans, contracts, architecture docs. No application code. |
| [**SmartProcurement-Backend**](https://github.com/MakarevichAV/SmartProcurement-Backend) | FastAPI API, domain logic, LORM enforcement, audit, jobs, worker. |
| [**SmartProcurement-Frontend**](https://github.com/MakarevichAV/SmartProcurement-Frontend) | React web UI for the Administrator, Buyer, and Approver roles. |

The `backend/` and `frontend/` directories inside this repo are those standalone repositories;
they are git-ignored here (no submodules).

```
SmartProcurement/
├── .specify/        Spec Kit config, constitution, templates, scripts
├── specs/001-smart-procurement/   spec.md · plan.md · research.md · data-model.md
│                                  contracts/ · quickstart.md · tasks.md
├── docs/            development-setup.md (+ more as phases land)
├── CLAUDE.md        orientation guide for AI agents in this repo
├── backend/         → SmartProcurement-Backend  (git-ignored)
└── frontend/        → SmartProcurement-Frontend (git-ignored)
```

## Implementation status

Delivered via `/speckit-implement` against
[`specs/001-smart-procurement/tasks.md`](specs/001-smart-procurement/tasks.md) (167 tasks, 12
phases):

| Phase | Scope | Status |
|-------|-------|--------|
| **Phase 1 — Setup** | Repo scaffolds, tooling, Docker Postgres, app skeletons | ✅ **complete** (T001–T010) |
| **Phase 2 — Foundational** | Identity/auth, RBAC, enterprise, capability registry + minimal promotion, audit + append-only guard, job queue/worker, AI provider seam, app shell | ✅ **complete** (T011–T038) |
| **Phase 3 — US1: Data sources & L0 map** | Connectors, field mappings, domain map | ⛔ **not started** |
| Phases 4–12 | Observation, risk/recommendation (L2/L3), L4 approval + execution, L5 policies, verification & demotion, audit UI, user admin, polish | ⛔ not started |

### What works today (Phase 1 + 2)

**Backend**

- FastAPI service with a unified error model and an `X-Correlation-Id` on every request; CORS
  configured for the SPA origin.
- **PostgreSQL + Alembic** — 3 migrations; `alembic check` reports no drift.
- **Authentication** — email/password (Argon2id), short-lived JWT access tokens, **rotating
  refresh tokens** (only their hash is stored; individually revocable) delivered as an
  **httpOnly cookie**; `POST /auth/login | /auth/refresh | /auth/logout`, `GET /me`.
- **RBAC** — an 11-permission catalogue mapped to roles (Administrator / Buyer / Approver);
  every route declares a required permission, enforced by the backend.
- **Enterprise boundary** — single active enterprise per deployment; `GET/PATCH /enterprise`.
- **Capability registry + LORM level enforcement** — the 8 procurement capabilities seeded at
  fixed levels with `l5_allowed` flags; `GET /capabilities`, `/capabilities/{key}/history`.
- **One-level promotion infrastructure** — `promotion_request` + a capability service that is
  the *only* writer of `capability.level`, enforcing human-only, exactly-one-step promotion
  and the "never L5 for a capped capability" rule; `POST /capabilities/{key}/promotion-requests`
  and `/promotion-requests/{id}/approve|reject`; append-only `capability_level_event` history.
- **Append-only audit infrastructure** — `audit_record` + an `AuditRecorder` that asserts the
  required field set, protected by a PostgreSQL `forbid_mutation()` trigger (UPDATE/DELETE
  raise) on `audit_record` and `capability_level_event`.
- **Durable job queue / worker foundation** — a `job` table drained with
  `FOR UPDATE SKIP LOCKED`, exponential backoff, recurring self-re-enqueue; a
  `python -m app.worker` loop with a `--run-once` mode.
- **AI provider abstraction** — `LLMProvider` protocol with an Anthropic implementation and a
  `DeterministicMockProvider`; a `generate_structured` wrapper that validates output against a
  schema and, on persistent failure, opens an observability gap + writes an `ai_unavailable`
  audit record instead of proceeding.
- Vendored LORM policy schema + validator (pinned) with a backend adapter.
- Demo seed CLI; 30 automated tests; ruff + black + `mypy --strict` clean.

**Frontend**

- Vite/React app **shell**: login screen, silent refresh on load, protected routing.
- **Login / logout**; **authenticated user profile** (name, roles, effective permissions) in
  the header; **navigation shell** with the nine product areas.
- Access token held **in memory only** (never in `localStorage`); errors surfaced as toasts
  from the backend's unified error model.
- OpenAPI type-generation workflow (`npm run gen:api`).

### What is *not* implemented yet (planned)

Everything past the foundation, including: data-source connectors and the L0 domain map;
the observation loop, risk detection and AI explanations (L2); recommendations and the
`allow / ask / deny` enforcement gate (L3); L4 approval + execution adapters; L5 autopilot
policies; verification and automatic demotion; audit-query endpoints; user-management
endpoints; and **every business screen in the UI** (Dashboard, Risks/Recommendations,
Approvals, Autopilot/Policies, Capabilities, Data Sources, Executions/Orders, Audit,
Users & Roles) — these are **placeholder pages** until their phase lands.

## Running locally

Full instructions, ports and env vars: [`docs/development-setup.md`](docs/development-setup.md).
Per-repo detail: [`backend/README.md`](backend/README.md), [`frontend/README.md`](frontend/README.md).

```sh
# 1. database
docker compose up -d postgres            # PostgreSQL 16, 127.0.0.1:5432 (loopback only)

# 2. backend  → http://localhost:8000  (/health, /docs, /openapi.json)
cd backend
uv sync --extra dev
cp .env.example .env                      # set FERNET_KEY (see the file)
uv run alembic upgrade head
uv run python -m app.seed --demo          # demo enterprise + users (LOCAL DEV ONLY)
uv run uvicorn app.main:app --reload
uv run python -m app.worker               # (separate shell) background worker

# 3. frontend → http://localhost:5173
cd ../frontend
npm install
cp .env.example .env
npm run dev
```

Demo login (local development only): `admin@example.com` / `buyer@example.com` /
`approver@example.com`, password `demo`. See `backend/README.md`.

## Documents

| Document | Purpose |
|----------|---------|
| [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | Project constitution (v1.0.0) — principles of responsibility, security, audit, LORM |
| [`specs/001-smart-procurement/spec.md`](specs/001-smart-procurement/spec.md) | Functional specification for v1 + Clarifications |
| [`specs/001-smart-procurement/plan.md`](specs/001-smart-procurement/plan.md) | Technical plan, Constitution Check, project structure |
| [`specs/001-smart-procurement/research.md`](specs/001-smart-procurement/research.md) | Technical decisions and rationale |
| [`specs/001-smart-procurement/data-model.md`](specs/001-smart-procurement/data-model.md) | Data model and state machines |
| [`specs/001-smart-procurement/contracts/`](specs/001-smart-procurement/contracts/) | Contracts: REST API, AI output, execution adapter, source connector, LORM enforcement |
| [`specs/001-smart-procurement/tasks.md`](specs/001-smart-procurement/tasks.md) | 167 dependency-ordered implementation tasks (Phases 1–2 complete) |
| [`specs/001-smart-procurement/quickstart.md`](specs/001-smart-procurement/quickstart.md) | End-to-end validation walkthrough (targets the full v1) |
| [`docs/development-setup.md`](docs/development-setup.md) | Local setup: three repos, branching, ports, commands |
| [`CLAUDE.md`](CLAUDE.md) | Orientation guide for working in this repository |

## Development workflow (Spec-Driven)

Governance artifacts are produced through Spec Kit from this repo root:

```
/speckit-constitution   → constitution               (done)
/speckit-specify        → feature specification       (done)
/speckit-clarify        → clarifications in the spec  (done)
/speckit-plan           → plan + contracts            (done)
/speckit-tasks          → tasks.md                    (done)
/speckit-analyze        → cross-artifact consistency  (done — clean)
/speckit-implement      → implementation from tasks.md  (Phases 1–2 complete; Phase 3 next)
```

Branching (all three repos): `feature/<slug>` for a phase, `fix/<slug>` for fixes, `main` as
the always-runnable integration branch; a cross-repo change uses the same branch name in
every affected repo.
