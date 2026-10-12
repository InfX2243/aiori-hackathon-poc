# 5. Roadmap, Demo, and Evaluation

## Phased implementation

### Phase 0 — Host and image validation
Record OS, architecture, kernel, CPU, RAM, and Cloud Hypervisor version. Check /dev/kvm and permissions on both hosts. Identify image format/checksum and required kernel, initrd, arguments, and network configuration. Manually boot and stop the same image on both hosts.

**Acceptance:** both hosts boot the prepared guest and clean up the process and socket reliably.

### Phase 1 — Verify Cloud Hypervisor API
Pin a release. Verify official API schema for configuration, boot, inspection, shutdown, and deletion. Use a private Unix socket. Observe process exit and cleanup.

**Acceptance:** repeatable single-host lifecycle; no invented fields or unverified CLI flags.

### Phase 2 — Single-host worker
Implement health/capacity, create/start/stop/list/detail, local policy, image allowlist, process supervision, child reaping, logs, and socket cleanup.

**Acceptance:** real lifecycle tests pass, including invalid input and controlled startup failure.

### Phase 3 — Controller and persistence
Implement public REST API, authentication/authorization, SQLite records, lifecycle transitions, idempotency, bounded worker client calls, and error classification.

**Acceptance:** API/state tests pass and metadata survives controller restart.

### Phase 4 — Second worker and placement
Deploy identical worker binary on Host 2. Register both workers, report capacity, add safety margins, host caps, and atomic reservations.

**Acceptance:** tests prove each host can be selected and insufficient capacity is rejected.

### Phase 5 — Failure recovery
Implement unknown/pending operations, worker/hypervisor crash detection, restart reconciliation, reservation repair, safe cleanup, concurrency and duplicate-request tests.

**Acceptance:** no false success on unknown outcomes, no duplicate VM after retry/restart, no unsafe reservation release.

### Phase 6 — Security and demo
Verify authentication and network isolation. Run functional, security, concurrency, and leak tests. Document deployment and rehearse with captured evidence.

**Acceptance:** another team member can reproduce the demo from repository instructions.

## Five-minute demo
1. Show controller health and both workers registered.
2. Show host capacity, timestamps, and inventory.
3. Create a VM and explain the selected host.
4. Start it; verify through controller, worker, and host process inspection.
5. Create a second VM and show reservations affecting placement.
6. Submit an over-limit request and show rejection.
7. Submit an insufficient-capacity request and prove no VM was created.
8. Restart controller or inject a controlled worker/hypervisor failure.
9. Show reconciliation or an explicit unknown state.
10. Stop a VM and verify process/socket cleanup and consistent accounting.

Use three terminal panes: API requests, controller/worker logs, and independent host inspection. Preflight the image and network.

## Memorable reliability demonstration
Start a VM, restart the controller while the VM remains running, then query inventory. Demonstrate that the controller rediscovers the existing worker-managed process instead of creating a duplicate. Claim success only after verifying actual process and guest state.

## Measurements
Collect placement decisions and reasons, create/start/stop latency, lifecycle success/failure counts, reservation accuracy, failure detection time, recovery time, leftover process/socket count, and concurrent reservation results. Do not fabricate benchmarks; preserve raw output and conditions.

## Technical report outline
1. Problem statement and goals.
2. Non-goals and assumptions.
3. Requirements matrix.
4. Architecture and responsibilities.
5. API and authentication.
6. Placement and reservation transaction.
7. Lifecycle and operation semantics.
8. Process supervision and cleanup.
9. Failure handling and reconciliation.
10. Test methods and actual results.
11. Security controls.
12. Limitations and future work.
13. Reproducible deployment/demo instructions.

## What distinguishes a strong submission
Correctness under concurrent requests; honest unknown outcomes; safe process ownership and teardown; memory admission with reservations; restart recovery; defense-in-depth validation; reproducible tests; measured results and explicit limitations.

## Immediate next action
Complete Phase 0 before writing a large codebase. Verify the exact image boots and shuts down on both hosts, then build a single-host worker vertical slice.
