# 3. Placement, Lifecycle, and Reliability

## Memory admission and placement

Use deterministic **most-available-admissible-memory** placement with atomic reservations.

For each host:
- A = Linux-reported available memory.
- S = configured safety reserve for OS, worker, and unrelated activity.
- R = memory reserved for managed VMs, including pending operations.
- M = requested VM memory.
- L = configured cap on managed reservations.

```text
C = max(0, min(A - S, L - R))
```

A host is eligible if healthy/reachable, below its VM-count cap, and C >= M. Select the eligible host with largest C; break ties by stable host ID.

This is an admission heuristic, not a guarantee against all memory pressure. Available memory changes, and memory-only placement does not guarantee CPU, disk, or network capacity.

### Worked example

| Measurement | Host 1 | Host 2 |
|---|---:|---:|
| Available A | 4096 MiB | 6144 MiB |
| Safety S | 1024 MiB | 1024 MiB |
| Existing/pending R | 2048 MiB | 1024 MiB |
| Reservation cap L | 6144 MiB | 8192 MiB |
| A - S | 3072 MiB | 5120 MiB |
| L - R | 4096 MiB | 7168 MiB |
| Admissible C | 3072 MiB | 5120 MiB |

A 2048 MiB VM fits both and selects Host 2. A 5500 MiB VM fits neither and is rejected without creation.

### Concurrent requests and stale capacity
Two requests may read the same capacity and both select one host. Use a SQLite write transaction to read reservations, select an eligible host, insert a pending reservation and operation record, then commit before dispatch. Later requests must account for the reservation.

If a worker times out, do not immediately release the reservation; first reconcile whether the VM exists or is running. A database transaction serializes controller decisions but cannot reserve physical RAM atomically on Linux.

Workers should report total and available memory plus observation timestamps. Reject or refresh stale reports. Apply configurable safety headroom.

## Lifecycle semantics
- **Create:** validate, reserve, prepare resources/configuration, and register a stopped definition. Do not boot.
- **Start:** boot a defined/stopped VM and verify runtime state.
- **Stop:** request supported graceful shutdown; wait a bounded period; escalate only under a documented policy and after verifying process ownership.
- **List:** return controller metadata with observed state and freshness/source.
- **Delete (optional):** delete only a stopped VM; remove managed resources and metadata after cleanup is verified.

Graceful guest shutdown, process exit, definition deletion, and resource cleanup are distinct events.

## State machine

Suggested visible states:
- Defined: definition exists and is not running.
- Starting: start operation in progress.
- Running: running state verified.
- Stopping: stop operation in progress.
- Stopped: not running; definition preserved.
- Failed: failure confirmed and recorded.
- Unknown: evidence insufficient or observations conflict.

Transitions:
- Create: absent → Defined.
- Start: Defined/Stopped → Starting → Running.
- Stop: Running → Stopping → Stopped.
- Confirmed start failure: Starting → Failed or Stopped, depending on cleanup evidence.
- Unexpected exit: Running → Failed or Stopped based on evidence/policy.
- Worker unreachable/conflicting evidence: current state → Unknown.
- Reconciliation moves Unknown only when sufficient evidence exists.

Reject conflicting operations with 409 or serialize them per VM. Repeated operations should be idempotent where safe.

## Idempotency and uncertain outcomes
Persist idempotency key, request hash, VM ID, and outcome. Same key plus same payload returns the existing result; same key with a different payload returns conflict. Start/stop operations need durable operation IDs.

Classify outcomes as:
- **Succeeded:** actual state verified.
- **Failed:** failure verified and cleanup/reservation outcome known.
- **Pending:** operation tracked and still in progress.
- **Unknown:** evidence cannot establish success or failure.

A timeout is not proof that the operation failed. Inspect worker inventory and hypervisor state before retrying.

## Process supervision and cleanup
The worker should:
1. Create private per-VM working directory and management socket.
2. Start Cloud Hypervisor using validated arguments/configuration; never interpolate client strings into shell commands.
3. Track process identity and wait/reap the child.
4. Verify the process belongs to the managed VM before signaling.
5. Attempt documented graceful shutdown.
6. Wait a bounded grace period.
7. Force-terminate only if policy permits and ownership is reverified.
8. Confirm exit and reap.
9. Remove only owned, inactive sockets/temp files.
10. Report cleanup errors and retain reservations if state remains uncertain.

Do not trust a bare PID because PIDs can be reused. Prefer supervised child relationships and process identity checks. Killing the worker does not guarantee children terminate. Verify exact shutdown behavior for the pinned Cloud Hypervisor version.

## Failure handling

| Scenario | Required behavior |
|---|---|
| Validation failure | Reject before dispatch; no resource leak |
| Worker unreachable before operation | Explicit unavailable response |
| Timeout after dispatch | Pending/unknown; inspect before retry/release |
| Hypervisor API error | Record failure; inspect process/resources |
| Hypervisor exits unexpectedly | Detect/reap; reconcile VM and reservation |
| Controller restart | Reload SQLite and reconcile; do not assume processes vanished |
| Worker restart | Rediscover processes/inventory or report Unknown |
| Cleanup error | Preserve evidence; no unsafe deletion/release |
| Duplicate create | Return existing result for same key/payload |
| No host capacity | Reject; do not create VM |
| Contradictory worker report | Mark Unknown/Reconciling and gather evidence |

## Reconciliation
Compare durable controller records, desired state, worker-local inventory, process ownership, Cloud Hypervisor observations, host health, and reservations. An unreachable worker is not proof that its VM stopped. Missing process evidence is not proof every resource was cleaned up.

## Example demo limits
These are adjustable examples, not universal defaults:
- VM memory: 256–2048 MiB.
- vCPUs: 1–2 per VM.
- Running VMs per host: at most 2.
- Safety reserve: 25% of physical RAM plus a minimum reserve chosen after host inspection.
- Controller operations in flight: 8.
- Worker operations in flight: 2 per host.
- JSON body: 16 KiB.
- Synchronous wait: 30 seconds; operation tracking continues beyond it.

Adjust after inspecting host capacity and guest requirements. Enforce in controller and worker.
