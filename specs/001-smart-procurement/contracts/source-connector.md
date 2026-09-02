# Contract — Source Connector

`integration/` reads enterprise data through swappable connectors. The domain model never
depends on a specific ERP (Constitution IX).

## Interface

```python
class SourceConnector(Protocol):
    connector_type: str  # "rest" | "file" | "sql"

    async def test_connection(self) -> ConnectionCheck: ...
    async def describe_schema(self) -> list[SourceField]: ...
    async def fetch(self, since: datetime | None) -> FetchBatch: ...
```

## Models

```
ConnectionCheck:
  health: enum(available, unavailable)
  detail: str
  checked_at: datetime

SourceField:
  path: str                 # "items[].sku", "STOCK.QTY_ON_HAND", column name, ...
  inferred_type: str
  sample_values: list[str]  # capped, PII-safe

FetchBatch:
  records: list[object]     # raw rows/objects, connector-native shape
  cursor: str | null        # opaque high-water mark for the next incremental fetch
  fetched_at: datetime
```

## v1 connectors

| type | config (non-secret) | credential (`secret`) | notes |
|---|---|---|---|
| `rest` | `base_url`, `resource_paths[]`, `pagination`, `incremental_param` | `{kind: bearer\|basic\|api_key, ...}` | JSON over HTTP |
| `file` | `format: csv\|json`, `delimiter`, `has_header` | — | operator uploads a file; stored, then parsed |
| `sql` | `dsn_without_credentials`, `queries: {entity: "SELECT ..."}` | `{username, password}` | **read-only**; backend rejects non-`SELECT` |

## Pipeline placement

1. `test_connection()` on connect and on a schedule → updates `data_source.health`
   (`available` / `stale` / `unavailable`) and opens/closes an `observability_gap`
   (FR-002/FR-011/FR-015).
2. `describe_schema()` → persists `source_field[]` → feeds AI `MappingSuggestion`
   (ai contract §1). Nothing is auto-applied (FR-004).
3. `fetch(since)` runs from the `observe_source` job at `data_source.observation_interval`.
   Raw records are mapped through **confirmed** `field_mapping` rows only into canonical
   `domain/` rows (with `source_provenance`), then diffed into `observation_signal` rows
   (FR-005/FR-007/FR-013/FR-014).
4. Connector errors → `AdapterError` → mapped to the unified error model; the source goes
   `unavailable`; dependent autonomous actions are blocked (FR-016).

## Guarantees

- No connector writes back to the source (read-only integration for v1).
- Credentials come only from the `SecretStore`; never logged, never sent to the LLM
  (FR-068).
- Adding a new connector = new class implementing this Protocol + a `connector_type` value;
  no change to `domain/`, `observation/`, or `analysis/` (SC-015).
