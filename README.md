# Smart Procurement

An intelligent decision layer for enterprise procurement. The system connects to corporate
data sources, builds a single picture of the procurement situation, detects and explains risks
(shortages, missed lead times, price and quality anomalies), produces buying recommendations,
and — depending on the authority level under the **LORM** model — shows the decision to a
human, asks for confirmation, or executes the action autonomously within an approved policy.

Smart Procurement **does not replace** ERP/WMS/MES. It is a decision layer on top of existing
systems; execution and bookkeeping stay with them.

## Architecture

Three separate Git repositories:

| Repository | Contents |
|------------|----------|
| **SmartProcurement** (this one) | Constitution, specifications, plans, contracts, architecture docs. No application code. |
| [**SmartProcurement-Backend**](https://github.com/MakarevichAV/SmartProcurement-Backend) | Python · FastAPI · PostgreSQL. REST API, domain logic, AI Decision Layer, LORM enforcement, policies, execution adapters, audit, background processing. Modular monolith + worker process. |
| [**SmartProcurement-Frontend**](https://github.com/MakarevichAV/SmartProcurement-Frontend) | React · TypeScript · Vite. Web UI for the Administrator, Buyer, and Approver roles. |

Backend and Frontend communicate only through a REST/OpenAPI contract. The `backend/` and
`frontend/` directories inside this repository are standalone repositories and are
deliberately git-ignored here (no submodules).

Layers: Frontend → Backend/API → AI Decision Layer → **LORM Responsibility/Enforcement** →
Data/Integration → Execution → Persistence. The backend is the only security boundary.

- Backend: <https://github.com/MakarevichAV/SmartProcurement-Backend>
- Frontend: <https://github.com/MakarevichAV/SmartProcurement-Frontend>

**Status**: Phase 1 (scaffolding) is implemented and runnable — FastAPI skeleton with
`/health` + OpenAPI, and a Vite/React shell. Phases 2+ (identity/auth, LORM, domain, …) are
in `specs/001-smart-procurement/tasks.md`.

## Role of LORM

The [Layered Operational Responsibility Model](https://github.com/Argyronix/lorm) assigns an
**authority level (L0–L5) to each capability individually**, not to the system as a whole:

- **L0–L2** — domain awareness, observation, diagnosis with a confidence estimate.
- **L3** — recommendation; the human decides.
- **L4** — an action is prepared and executed only after per-action human approval.
- **L5** — autonomous execution within the bounds of a machine-readable policy that has a
  distinct author and approver.

LORM governs authority, not the procurement decision itself. Level promotion is human-only and
one step at a time; demotion is automatic on policy violation, verification failure, loss of
observability, or policy expiry. The official policy schema and validator are reused as-is;
enforcement is adapted to the backend runtime — see
[`contracts/lorm-enforcement.md`](specs/001-smart-procurement/contracts/lorm-enforcement.md)
and [`research.md` §6](specs/001-smart-procurement/research.md).

## Documents

| Document | Purpose |
|----------|---------|
| [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | Project constitution (v1.0.0) — principles of responsibility, security, audit, LORM |
| [`specs/001-smart-procurement/spec.md`](specs/001-smart-procurement/spec.md) | Functional specification for v1 + Clarifications |
| [`specs/001-smart-procurement/plan.md`](specs/001-smart-procurement/plan.md) | Technical plan, Constitution Check, project structure |
| [`specs/001-smart-procurement/research.md`](specs/001-smart-procurement/research.md) | Technical decisions and rationale |
| [`specs/001-smart-procurement/data-model.md`](specs/001-smart-procurement/data-model.md) | Data model and state machines |
| [`specs/001-smart-procurement/contracts/`](specs/001-smart-procurement/contracts/) | Contracts: REST API, AI output, execution adapter, source connector, LORM enforcement |
| [`specs/001-smart-procurement/tasks.md`](specs/001-smart-procurement/tasks.md) | 167 dependency-ordered implementation tasks, grouped by user story |
| [`specs/001-smart-procurement/quickstart.md`](specs/001-smart-procurement/quickstart.md) | End-to-end validation walkthrough |
| [`docs/development-setup.md`](docs/development-setup.md) | Local setup: three repos, ports, run commands |
| [`CLAUDE.md`](CLAUDE.md) | Orientation guide for working in this repository |

## Development (Spec-Driven)

Work is done through Spec Kit from the repository root:

```
/speckit-constitution   → constitution               (done)
/speckit-specify        → feature specification       (done)
/speckit-clarify        → clarifications in the spec  (done)
/speckit-plan           → plan + contracts            (done)
/speckit-tasks          → tasks.md                    (done)
/speckit-analyze        → cross-artifact consistency  (done — clean)
/speckit-implement      → implementation from tasks.md  (in progress — Phase 1 / T001–T010 done)
```

## Running locally

Full instructions and ports are in [`docs/development-setup.md`](docs/development-setup.md).

**Phase 1 scaffold** (no database needed yet):

```sh
# backend/  (Python 3.12 + uv)  → http://localhost:8000/health , /openapi.json
cd backend && uv sync --extra dev && uv run uvicorn app.main:app --reload

# frontend/ (Node 20+)          → http://localhost:5173
cd frontend && npm install && npm run dev
```

**Later phases** additionally need PostgreSQL and the worker:

```sh
docker compose up -d postgres            # local Postgres 16 (loopback only)
cd backend && uv run alembic upgrade head && uv run python -m app.seed --demo
uv run python -m app.worker              # scheduler + durable job queue
```

The end-to-end demo walkthrough lives in
[`quickstart.md`](specs/001-smart-procurement/quickstart.md).
