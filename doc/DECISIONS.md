# Decision Log

Update this log whenever evidence or constraints change.

## D-001 — Identical worker agents
**Status:** Recommended. Use the same implementation and config schema on both hosts to reduce divergence.

## D-002 — Go
**Status:** Provisional. Go offers standard HTTP support, concurrency primitives, and simple binary deployment. Revisit if team experience or schedule strongly favors Python or Rust.

## D-003 — SQLite
**Status:** Recommended. Persist VM metadata, operations, idempotency records, and reservations. It survives controller restarts and supports transactions without a separate database service. Transactions serialize controller decisions, not physical memory allocation.

## D-004 — Create does not boot
**Status:** Provisional. Create prepares a stopped VM definition; start boots it. Revisit if the evaluator defines create as immediate boot.

## D-005 — Stop preserves definition
**Status:** Recommended. Stop the guest but retain its definition for restart; deletion is separate.

## D-006 — Synchronous calls plus operation tracking
**Status:** Recommended. Start with synchronous HTTP, durable operation IDs, and pending/unknown outcomes after bounded waits. Add a queue only if required.

## D-007 — Memory admission and reservations
**Status:** Recommended. Use most-available-admissible-memory, safety headroom, per-host caps, and atomic reservation transactions. Memory alone cannot guarantee CPU, disk, network, or all memory safety.

## D-008 — Local hypervisor management socket
**Status:** Recommended; verify version-specific details. Keep the management endpoint local to the worker with restrictive permissions.

## D-009 — Unknown is a valid outcome
**Status:** Recommended. A timeout or conflicting observation must trigger inspection/reconciliation, not an invented success/failure.

## D-010 — Avoid unnecessary infrastructure
**Status:** Recommended. No Kubernetes, service mesh, consensus, distributed storage, or message broker without a demonstrated requirement.

## Open verification items
- [ ] Record OS, kernel, architecture, CPU virtualization, RAM, and /dev/kvm permissions on both hosts.
- [ ] Record exact Cloud Hypervisor version and verify official API schema.
- [ ] Confirm image format/checksum, boot inputs, and networking.
- [ ] Demonstrate manual boot, graceful shutdown, child reaping, and socket cleanup on both hosts.
- [ ] Choose shared-secret-over-TLS or mTLS based on network/certificate setup.
- [ ] Confirm create semantics and deletion expectations.
- [ ] Record actual test outcomes before labeling anything Tested.
