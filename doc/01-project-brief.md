# 1. Project Brief and Requirements

## Problem statement
Build a small central controller and two worker agents on two KVM-capable hosts. The controller exposes an authenticated API to create, start, stop, and list VMs through Cloud Hypervisor. It places VMs based on available host memory, enforces resource limits, handles failures, and prevents orphaned VM processes.

The goal is a small, defensible virtualization control plane—not a general-purpose cloud platform.

## Why a controller?
Each host can run VMs independently, but without a controller users must know which host to manage, track capacity manually, and coordinate state. A controller provides one management entry point, consistent policy, placement, inventory, and recovery metadata. It does not make two hosts one machine; host failure, network partitions, stale observations, and uncertain outcomes still exist.

## Core concepts
### Cloud Hypervisor and KVM
Cloud Hypervisor is the userspace virtual machine monitor (VMM) that configures and runs a guest and exposes a management API. Linux KVM provides kernel virtualization facilities through /dev/kvm. The binary, kernel, CPU capabilities, permissions, and guest configuration must be compatible.

### VM image and boot inputs
A disk image may contain a guest OS or filesystem, but boot may also require a kernel, initrd, boot arguments, firmware, and device configuration depending on boot mode. Record the exact image, format, checksum, and required inputs.

### Worker agent
A host-local service that authenticates controller requests, validates host-local policy, reports capacity and inventory, starts and supervises Cloud Hypervisor, uses its local management API, reaps child processes, and cleans up sockets/files. Both hosts should use the same implementation with host-specific protected configuration.

### Placement and capacity
Placement selects a host for a VM. Memory is the required scheduling signal, but memory-only scheduling cannot guarantee CPU, disk, or network capacity. Host-reported available memory is an observation, not a reservation.

### Lifecycle and process ownership
- **Create:** prepare/register a VM definition and resources; by provisional convention, do not boot.
- **Start:** boot a defined stopped VM.
- **Stop:** shut down the guest while preserving its definition for restart.
- **Delete:** remove a stopped VM definition and managed resources; optional unless required.
- **Terminate process:** end a process; not equivalent to graceful guest shutdown.
- **Cleanup:** remove owned resources and release reservations only when safe.

An orphan is a process no longer correctly managed. Leaks can include processes, sockets, temporary files, and reservations. Killing a worker does not guarantee its children terminate.

### Security and failures
Authentication identifies the caller; authorization controls permitted actions. A timeout bounds waiting but does not prove remote failure. Retries can duplicate work. The controller must distinguish verified success, verified failure, and unknown outcomes requiring inspection.

## Scope
### Explicit requirements
- Two hosts with working /dev/kvm and preconfigured networking.
- One prepared image.
- One controller and two worker agents.
- VM create, start, stop, and list.
- Authenticated controller-to-worker communication.
- Memory-based placement and resource limits.
- Failure handling and cleanup without orphaned VM processes.
- Lifecycle and two-host placement tests.
- Technical report and demonstration.

### Provisional assumptions
- Create registers a stopped VM definition; start boots it.
- Stop preserves the definition.
- VM IDs are generated centrally and stable across retries.
- Controller owns desired state, placement, durable metadata, and global reservations.
- Workers own local process identity and runtime observations.
- SQLite is sufficient for single-controller persistence.
- Hypervisor management endpoints remain local to workers.
- Uncertain operations remain pending/unknown until reconciled.

### Optional enhancements
Delete endpoint, richer operation history, mutual TLS, metrics, terminal dashboard, and periodic reconciliation.

### Out of scope
Live migration, distributed storage, browser VNC, image marketplace, new hypervisor, Kubernetes, service mesh, consensus, or separate message broker without a demonstrated need.

## Requirements matrix

| Requirement | Expected behavior | Owner | Approach | Verification | Importance |
|---|---|---|---|---|---|
| Two KVM hosts | Both boot the guest | Host setup/worker | Validate kernel, CPU, permissions | Manual boot each host | Critical |
| Prepared image | Repeatable boot | Worker/config | Pin image and boot inputs | Repeated boot test | Critical |
| Authenticated API | Protected management operations | Controller | REST auth and validation | API tests | Critical |
| Two workers | Same agent on each host | Workers | Shared binary, host config | Health checks | Critical |
| Lifecycle | Create/start/stop/list work | Controller + worker | Explicit state machine | Integration tests | Critical |
| Worker authentication | Reject unauthorized callers | Worker | TLS and credential validation | Negative auth tests | Critical |
| Memory placement | Only eligible hosts selected | Controller | Admission formula and reservations | Scheduler tests | Critical |
| Resource limits | Invalid requests rejected | Both | Controller and worker validation | Boundary tests | Critical |
| Failure handling | Unknown outcomes represented honestly | All | Deadlines and reconciliation | Failure injection | Critical |
| Clean teardown | No managed process/socket leaks | Worker | Supervision and verified cleanup | Process/socket inspection | Critical |
| Restart recovery | Running VMs rediscovered | Controller + worker | SQLite and reconciliation | Restart test | High |
| Duplicate protection | Retry does not duplicate VM | Controller | Idempotency records | Duplicate-request test | High |
| Report/demo | Results reproducible | Team | Test artifacts and commands | Rehearsal | High |

## Open questions
1. What OS, kernel, CPU architecture, RAM, and Cloud Hypervisor version are installed on each host?
2. Has the exact image booted with Cloud Hypervisor? What kernel/initrd/arguments are needed?
3. Is the worker network private and encrypted, and how will credentials be provisioned?
4. Must real VMs run on both hosts for evaluation?
5. How much time remains, and which language does the team know best?

## Initial success definition
Create a VM definition, start it on an eligible host, inspect it, stop it, list it, reject invalid resource requests, demonstrate two-host placement, and prove failed operations do not leave unmanaged processes. Recovery and concurrency tests elevate the prototype to a robust systems submission.
