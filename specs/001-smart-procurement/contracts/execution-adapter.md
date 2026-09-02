# Contract — Execution Adapter

`execution/` dispatches an ERP-independent `ProcurementCommand` through a swappable adapter.
The decision layer imports **none** of this module (Constitution VIII).

## Interface

```python
class ExecutionAdapter(Protocol):
    adapter_type: str  # "simulated" | "generic_rest" | (future) "erp_x" | "rpa"

    async def dispatch(self, command: ProcurementCommand,
                       idempotency_key: str) -> DispatchResult: ...

    async def get_status(self, idempotency_key: str) -> ExecutionStatus: ...
```

- `dispatch` MUST be idempotent w.r.t. `idempotency_key`: a repeat call for a key the
  downstream system already accepted returns the original result, never a second order.
- `get_status` is used by `reconcile_executions` after a crash for actions left
  `sent_unknown` (FR-032a). It never causes a side effect.
- Adapter/transport errors are raised as `AdapterError(code, message, retryable: bool)` and
  mapped by `execution/` into the unified API error model (502/504).

## Models

```
ProcurementCommand:
  action_type: "create_po"
  item: { sku: str, name: str }
  supplier: { code: str, name: str }
  quantity: number
  unit_price: number
  currency: str
  needed_by: date
  metadata: { procurement_action_id: uuid, enterprise_id: uuid, capability_key: str }

DispatchResult:
  accepted: bool
  external_ref: str | null          # downstream PO id when known synchronously
  raw: object                       # adapter-specific echo, stored in execution_attempt.response

ExecutionStatus:
  state: enum(unknown, pending, executed, failed)
  external_ref: str | null
  detail: object
```

## v1 implementations (FR-031a)

### `SimulatedExecutionAdapter` — demo default, deterministic
- Config: `{ latency_ms, failure_rate, always_external_ref_prefix }` (tests set these).
- Keeps an in-DB `simulated_order` table keyed by `idempotency_key` so repeat `dispatch`
  and `get_status` are truly idempotent and survive restart.
- Default config: 0 failures, small latency → the primary procurement scenario is repeatable.

### `GenericRestExecutionAdapter` — real HTTP integration
- Config (from `enterprise` + a `secret` of purpose `execution_endpoint`):
  `{ base_url, method, path_template, headers, auth: {kind, secret_ref}, timeout_s,
     idempotency_header: "Idempotency-Key", status_path_template }`.
- `dispatch` → `POST {base_url}{path}` with the command as JSON and the idempotency key in
  the configured header. 2xx → `accepted=true` (+`external_ref` from a JSONPath in config).
  4xx → `AdapterError(retryable=false)`; 5xx / network → `AdapterError(retryable=true)`.
- `get_status` → `GET {status_path}` mapped to `ExecutionStatus`.
- Verified in tests against a configurable stand-in HTTP service (proves swappability, SC-014).

## Selection

`enterprise.active_execution_adapter` chooses the adapter at dispatch time. Switching it
requires **no change** to `decision/`, `lorm/`, or `verification/` (SC-014 test).

## Guard

`execution.dispatch(action, decision: EnforcementDecision)` refuses to run unless `decision`
is an `allow` produced by `LormEnforcementService` for this exact `procurement_action`
(capability + params hash). There is no other code path to an adapter (FR-069).
