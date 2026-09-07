# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **root governance repository** of the Smart Procurement project (a
[Spec Kit](https://github.com/github/spec-kit) SDD workspace). It holds the constitution,
specifications, plans, tasks, contracts, and architecture docs — **not application code**.

Per `specs/001-smart-procurement/plan.md` §1 the code lives in two *separate* git repos that
sit inside this directory and are meant to be git-ignored by the root:

| Path | Repo | Stack (planned) |
|------|------|-----------------|
| `./` | `SmartProcurement` | Spec Kit artifacts, `docs/`, README |
| `./backend/` | `SmartProcurement-Backend` | Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0 async, Alembic, PostgreSQL 16 |
| `./frontend/` | `SmartProcurement-Frontend` | React 18, TypeScript, Vite, Redux Toolkit + RTK Query, Tailwind |

`backend/` and `frontend/` are working apps — **Phase 1 (Setup) and Phase 2 (Foundational)
are complete; Phase 3 has not started** (see `specs/001-smart-procurement/tasks.md` and the
root `README.md` for the implemented-vs-planned breakdown). They are **not** tracked from the
root repo — no submodules; the root `.gitignore` excludes `backend/`, `frontend/`, and
`.idea/`. Frontend and backend communicate only through the REST/OpenAPI contract in
`specs/001-smart-procurement/contracts/rest-api.md`.

**Branching** (all three repos, see `docs/development-setup.md`): `feature/<slug>` for new
functionality / spec phases, `fix/<slug>` for bug fixes, `main` as the always-runnable
integration branch. A cross-repo change uses the **same branch name** in every affected repo.
Branch off `main`; don't commit straight to `main`.

## Spec-driven workflow (this repo's "commands")

There is no build/lint/test here. Work happens through Spec Kit skills, invoked as slash
commands, in dependency order:

```
/speckit-constitution   → .specify/memory/constitution.md   (done: v1.0.0)
/speckit-specify        → specs/<NNN-slug>/spec.md           (done: 001-smart-procurement)
/speckit-clarify        → appends ## Clarifications to spec.md   (done)
/speckit-plan           → plan.md + research.md + data-model.md + contracts/ + quickstart.md (done)
/speckit-tasks          → tasks.md                           (done: 167 tasks, 12 phases)
/speckit-analyze        → cross-artifact consistency check (spec ↔ plan ↔ tasks)  (done — clean)
/speckit-implement      → executes tasks.md   (Phases 1–2 complete; Phase 3 next)
```

The active feature directory is stored in `.specify/feature.json` (git-ignored, per-checkout
state). Helper scripts under `.specify/scripts/bash/` are called by the skills, not usually by
hand; the useful one to run directly is:

```sh
.specify/scripts/bash/check-prerequisites.sh --json --paths-only   # resolve FEATURE_DIR / SPEC / PLAN / TASKS paths
.specify/scripts/bash/resolve-template.sh <template-name> --json    # e.g. spec-template, plan-template
```

Feature numbering is `sequential` (`NNN-slug`); scripts are `sh` (bash); the only integration
installed is `claude`.

## The governing model: LORM + Constitution

Every design and implementation decision is constrained by two documents. Read them before
changing anything in `specs/`:

- **`.specify/memory/constitution.md`** — 12 principles. The decisive one:
  *when demo convenience conflicts with responsibility / security / audit / LORM, the latter
  win.* The constitution supersedes other practices; changing it requires a version bump and
  an approver who is not the author.
- **LORM (Layered Operational Responsibility Model)** — the reference implementation at
  <https://github.com/Argyronix/lorm> is normative. `research.md` §6 and
  `contracts/lorm-enforcement.md` define exactly what is **reused unmodified** (the policy
  JSON schema + `validate_policy.py`), what is **adapted** (enforcement as an in-process
  FastAPI service instead of Claude-Code hooks), and what is **built here**.

LORM invariants that shape the data model and API (see `data-model.md`, `contracts/`):

- Autonomy level **L0–L5 is per capability**, never system-wide. The 8 procurement
  capabilities are `proc.{inventory.observe, demand.observe, risk.diagnose, order.recommend,
  po.create, replenish.routine, supplier.add, payment.release}`.
- `proc.supplier.add` and `proc.payment.release` are **hard-capped below L5** (a system must
  never widen its own authority; payments need segregation of duties).
- `capability.level` is only ever **raised by a human, one level at a time** (via an approved
  `promotion_request`); it is only ever **lowered automatically, one level**, with no human or
  AI approval, on the SPEC §6.3 triggers (incident/rollback, material verification failure,
  observability loss, uncertainty over threshold, policy expiry/revocation).
- **L5 execution requires an active machine-readable policy** whose `author_id != approved_by_id`
  — enforced regardless of the user's roles. AI may draft a policy, never approve or activate one.
- Before any state-changing action, `LormEnforcementService.evaluate()` returns
  `allow | ask | deny`; `execution.dispatch()` refuses to run without a matching `allow`
  decision token. There is no other path to an execution adapter.
- Every L4/L5 action produces an append-only `audit_record` with the LORM §10.3 field set
  *before* its effects count as complete. Audit is a first-class subsystem, independent of any
  LLM context.

## Backend architecture (foundation built in Phase 2; later modules still planned)

- **Modular monolith** `backend/src/app/<module>/` (each module owns its tables + a service
  interface) **plus a second process** `python -m app.worker` sharing the same code/DB. The
  modules map 1:1 to the constitution's layers (`api/`, `analysis/`+`ai/` = AI Decision Layer,
  `lorm/`+`policies/` = Enforcement, `integration/`+`domain/`+`observation/` = Data, `execution/`,
  `audit/` + models = Persistence). Modules talk through services, not each other's tables.
- **PostgreSQL only.** Durable background jobs are a single `job` table drained with
  `FOR UPDATE SKIP LOCKED`; recurring observation jobs re-enqueue themselves. `pgvector` is
  enabled but used *only* for semantic search over unstructured supplier/quality/document text.
- **LLM behind a seam.** `LLMProvider.generate_structured(schema, …) -> validated Pydantic
  model`; a real provider + a `DeterministicMockProvider` for the default test suite. Invalid
  / timed-out / unavailable output is handled identically: retry/backoff → mark
  "human-required" → block dependent L5 → audit. The LLM never triggers execution and never
  writes `capability.level`. Deterministic preprocessing/aggregation runs *before* any LLM
  call — never one call per raw observation signal.
- **Swappable edges** (Protocols): `ExecutionAdapter` (v1: `simulated` default + `generic_rest`),
  `SourceConnector` (v1: `rest` / `file` / `sql`), `SecretStore`, `AuthProvider`. Decision code
  imports none of the adapter/connector modules.
- Auth: JWT (Argon2id passwords) + RBAC via FastAPI dependencies; the **backend is the only
  security boundary** — frontend and LLM carry no trusted authz/LORM logic. External-source
  credentials are Fernet-encrypted at rest, never in prompts, never returned by the API.

## Deliberate "do NOT add without proven need"

From `plan.md` / `research.md` — these are decisions, not omissions:

- No microservices, no message broker (Kafka / RabbitMQ / Redis-as-broker).
- No agent framework (LangChain / LangGraph) — orchestration is plain Python.
- No separate vector database.
- No ERP-vendor-specific coupling in the domain model; no RPA adapter in v1.

If a task seems to require one of these, surface the trade-off against the constitution before
proceeding rather than adding it silently.

## Editing spec artifacts

- Keep `spec.md`, `plan.md`, `data-model.md`, and `contracts/` mutually consistent — a change
  in one usually needs matching edits in the others; run `/speckit-analyze` after larger edits.
- Requirements use RFC-2119 MUST/SHOULD/MAY and carry stable ids (`FR-xxx`, `SC-xxx`); new
  clarifications are appended under `## Clarifications` with a dated session heading, never
  interleaved.
- `specs/001-smart-procurement/checklists/requirements.md` has its own lifecycle owned by
  `/speckit-specify` and `/speckit-clarify`; `/speckit-implement` reads its checkbox state as
  a gate.
