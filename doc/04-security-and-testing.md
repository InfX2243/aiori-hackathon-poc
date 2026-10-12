# 4. Security and Testing

## Trust boundaries
1. Client → public controller API.
2. Controller → worker API over management network.
3. Worker → local Cloud Hypervisor management endpoint.
4. Worker → Linux process, filesystem, and KVM resources.

Never expose a privileged hypervisor endpoint to clients. Prefer a local Unix socket with strict permissions. Bind worker APIs to a private management interface and restrict access with firewall rules.

## Authentication and authorization
For the prototype, authenticate all VM management operations. Use high-entropy worker credentials over TLS; keep secrets out of source control and logs. Restrict secret files and process permissions. Authorize operations even if the initial deployment has one administrator role. Consider mTLS if certificate provisioning fits the schedule.

A shared secret is simpler but offers weaker per-worker identity and coarser rotation than mTLS. Document the trust model and rotate exposed credentials.

## Input and resource security
- Accept allowlisted image IDs, not arbitrary host paths.
- Validate IDs, names, schema, integer bounds, and resource limits.
- Never concatenate client input into shell commands.
- Use argument arrays/direct process APIs.
- Use private per-VM directories with safe ownership and permissions.
- Limit request size, concurrent operations, resources, VM count, and duration.
- Validate limits in controller and worker.
- Avoid logging credentials.
- Never signal based on an unverified PID.
- Protect socket paths from symlink/ownership mistakes.
- Use the narrowest practical OS privileges consistent with KVM and file access.

Reference: OWASP REST Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html

## Security checklist
- [ ] Missing/invalid client credentials rejected.
- [ ] Unauthorized client cannot create/start/stop VMs.
- [ ] Worker rejects unauthenticated controller calls.
- [ ] Hypervisor management socket not reachable by untrusted clients.
- [ ] Arbitrary image paths rejected.
- [ ] Oversized body and invalid JSON rejected.
- [ ] Out-of-range memory/vCPU/count rejected by both layers.
- [ ] Operation concurrency enforced.
- [ ] Secrets absent from logs.
- [ ] Cleanup cannot signal unrelated processes.
- [ ] Worker endpoint inaccessible from untrusted interfaces.

## Test layers
- **Developer machine:** unit/fake-worker tests for validation, transitions, placement, transactions, idempotency, API errors, authentication, and reconciliation.
- **Two hosts:** actual Cloud Hypervisor boot/stop, network authentication, placement, process cleanup, and restart recovery.

Do not present simulated tests as proof of real KVM behavior.

## Test matrix

| Test | Setup/action | Expected result | Verification |
|---|---|---|---|
| Create | Valid definition | Definition created, not booted | API and worker inventory |
| Start/stop | Run lifecycle on real VM | Verified running then stopped | API, worker, process state |
| List/detail | Query inventory | Consistent IDs, host, config, state | Compare controller/worker |
| Invalid request | Bad JSON/missing fields | 400/422, no resources | API and filesystem |
| Resource limits | Excess memory/vCPU | Rejected before launch | API/process list |
| Host 1 only eligible | Simulate Host 2 ineligible | Host 1 selected | Scheduler unit test |
| Host 2 only eligible | Reverse eligibility | Host 2 selected | Scheduler unit test |
| Both eligible | Known capacities | Largest admissible capacity selected | Scheduler test |
| No capacity | Both below threshold | Rejected, no VM created | API/inventory |
| Stale capacity | Age report past threshold | Refresh or mark ineligible | Unit/integration test |
| Concurrent creates | Submit simultaneously | Reservations stay within policy | DB and race test |
| Worker unreachable | Stop/restrict endpoint | Explicit unavailable/unknown | Logs/operation record |
| Worker API error | Inject error response | No false success | Integration test |
| Start timeout | Delay response after dispatch | Pending/unknown; reconcile before retry | Operation API/inventory |
| Hypervisor start failure | Inject controlled failure | Failure recorded, cleanup attempted | Process/socket inspection |
| Unexpected exit | Terminate managed child | Exit detected/reaped; state reconciled | Logs/process inspection |
| Controller restart | Restart with VM running | VM rediscovered; no duplicate | Before/after inventory |
| Worker restart | Restart worker with VM running | Rediscover or mark Unknown | Host process and inventory |
| Cleanup failure | Inject filesystem/process error | Error retained; no unsafe release | Logs, DB, filesystem |
| Repeated start/stop | Repeat requests | Safe idempotency or conflict | API/history |
| Duplicate create | Same key/payload twice | Same result, no duplicate process | DB/process list |
| Bad credentials | Missing/invalid credentials | 401/403 | API response |
| Direct management access | Attempt untrusted access | Not reachable | Network/socket checks |
| Process leak | Failed start and stop | No unmanaged hypervisor children | Process tree |
| Socket cleanup | Complete teardown | No stale managed sockets | Filesystem |
| Child reaping | Stop test child | No zombie child | Process state |
| PID safety | Cleanup while unrelated process exists | Unrelated process survives | Process identity |
| Reservation recovery | Fail/restart operation | Release only after safe confirmation | DB vs actual state |

## Evidence to retain
For each test, record host OS/kernel and Cloud Hypervisor version, exact command/request, expected and actual result, status/exit code, sanitized logs, independent verification, and limitations.

## Authoritative references
- Cloud Hypervisor: https://github.com/cloud-hypervisor/cloud-hypervisor
- Linux KVM: https://www.kernel.org/doc/html/latest/virt/kvm/index.html
- OWASP REST security: https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- Go: https://go.dev/doc/
- SQLite transactions: https://www.sqlite.org/lang_transaction.html

Pin a version and verify version-sensitive claims against the actual binary and hosts. If unclear, run a small verification experiment and record it.
