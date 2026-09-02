# Smart Procurement

Интеллектуальный слой поддержки и автоматизации закупочных решений предприятия.
Система подключается к корпоративным источникам данных, строит единую картину закупочной
ситуации, обнаруживает и объясняет риски (дефицит, срыв сроков, аномалии цен и качества),
формирует закупочные рекомендации и — в зависимости от уровня полномочий по модели **LORM** —
показывает решение человеку, запрашивает подтверждение или выполняет действие автономно в
рамках утверждённой policy.

Smart Procurement **не заменяет** ERP/WMS/MES. Это слой принятия решений над существующими
системами; исполнение и учёт остаются за ними.

## Архитектура

Три отдельных Git-репозитория:

| Репозиторий | Содержимое |
|-------------|------------|
| **SmartProcurement** (этот) | Конституция, спецификации, планы, контракты, архитектурная документация. Кода приложения не содержит. |
| **SmartProcurement-Backend** | Python · FastAPI · PostgreSQL. REST API, доменная логика, AI Decision Layer, LORM enforcement, policies, execution adapters, audit, фоновая обработка. Модульный монолит + worker-процесс. |
| **SmartProcurement-Frontend** | React · TypeScript · Vite. Веб-интерфейс для ролей Administrator, Buyer, Approver. |

Backend и Frontend взаимодействуют только через REST/OpenAPI-контракт. Каталоги `backend/` и
`frontend/` внутри этого репозитория — самостоятельные репозитории и намеренно исключены из
отслеживания через `.gitignore` (без submodules).

Слои: Frontend → Backend/API → AI Decision Layer → **LORM Responsibility/Enforcement** →
Data/Integration → Execution → Persistence. Backend — единственная граница безопасности.

- Frontend: `<url SmartProcurement-Frontend>`
- Backend: `<url SmartProcurement-Backend>`

## Роль LORM

[Layered Operational Responsibility Model](https://github.com/Argyronix/lorm) задаёт **уровень
полномочий (L0–L5) для каждой capability отдельно**, а не для системы в целом:

- **L0–L2** — знание предметной области, наблюдение, диагностика с оценкой уверенности.
- **L3** — рекомендация; решение принимает человек.
- **L4** — действие готовится и выполняется только после подтверждения человеком.
- **L5** — автономное выполнение в границах машиночитаемой policy с отдельными автором и
  утверждающим.

LORM отвечает за полномочия, а не за само закупочное решение. Повышение уровня — только
решением человека и на один шаг; понижение — автоматически при нарушении policy, сбое
verification, потере наблюдаемости или истечении policy. Официальные схема policy и валидатор
переиспользуются как есть; enforcement адаптирован под backend-рантайм — см.
[`contracts/lorm-enforcement.md`](specs/001-smart-procurement/contracts/lorm-enforcement.md)
и [`research.md` §6](specs/001-smart-procurement/research.md).

## Документы

| Документ | Назначение |
|----------|------------|
| [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | Конституция проекта (v1.0.0) — принципы ответственности, безопасности, аудита, LORM |
| [`specs/001-smart-procurement/spec.md`](specs/001-smart-procurement/spec.md) | Функциональная спецификация v1 + Clarifications |
| [`specs/001-smart-procurement/plan.md`](specs/001-smart-procurement/plan.md) | Технический план, Constitution Check, структура проекта |
| [`specs/001-smart-procurement/research.md`](specs/001-smart-procurement/research.md) | Технические решения и обоснования |
| [`specs/001-smart-procurement/data-model.md`](specs/001-smart-procurement/data-model.md) | Модель данных, машины состояний |
| [`specs/001-smart-procurement/contracts/`](specs/001-smart-procurement/contracts/) | Контракты: REST API, AI-выход, execution adapter, source connector, LORM enforcement |
| [`specs/001-smart-procurement/quickstart.md`](specs/001-smart-procurement/quickstart.md) | Сквозной сценарий проверки |
| [`CLAUDE.md`](CLAUDE.md) | Ориентир для работы в репозитории |

## Разработка (Spec-Driven)

Работа ведётся через Spec Kit из корня репозитория:

```
/speckit-constitution   → конституция              (готово)
/speckit-specify        → спецификация фичи         (готово)
/speckit-clarify        → уточнения в спецификации  (готово)
/speckit-plan           → план + контракты          (готово)
/speckit-tasks          → tasks.md                  ← следующий шаг
/speckit-implement      → реализация по tasks.md
```

## Локальный запуск

Появится после реализации backend и frontend. Ориентировочно (см.
[`quickstart.md`](specs/001-smart-procurement/quickstart.md)):

```sh
# backend/  (Python 3.12, PostgreSQL 16)
alembic upgrade head && python -m app.seed --demo
uvicorn app.main:app --reload      # API + OpenAPI
python -m app.worker               # планировщик и очередь фоновых задач

# frontend/ (Node 20)
npm install && npm run dev
```
