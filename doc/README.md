# PS-023: Minimal Two-Node Cloud Hypervisor VM Controller

This directory is the living engineering notebook for the hackathon project.

## Document map
- [Project brief and requirements](01-project-brief.md)
- [Architecture and API](02-architecture-and-api.md)
- [Placement, lifecycle, and reliability](03-placement-lifecycle-reliability.md)
- [Security and testing](04-security-and-testing.md)
- [Roadmap and demo](05-roadmap-and-demo.md)
- [Decision log](DECISIONS.md)

## Current recommendation
- One central controller and two identical worker agents.
- Go as the provisional implementation language.
- SQLite for durable controller metadata, operations, and atomic reservations.
- Cloud Hypervisor management API accessed locally by each worker, preferably through private Unix sockets.
- Deterministic memory-based placement with safety headroom and atomic reservations.
- Explicit lifecycle states, bounded timeouts, idempotency, and reconciliation after uncertain outcomes.
- No live migration, distributed storage, browser VNC, image marketplace, new hypervisor, Kubernetes, service mesh, consensus system, or message broker unless requirements change.

These are recommendations, not claims that implementation or tests have succeeded.

## Evidence conventions
- **Requirement:** explicitly stated by the challenge.
- **Decision:** agreed design choice.
- **Assumption:** provisional choice pending confirmation.
- **Verify:** check official documentation or actual host behavior.
- **Tested:** use only after a reproducible test passes; record environment and result.
- **Out of scope:** explicitly excluded.

## First milestone
Before implementing the controller, verify on both intended hosts that the exact prepared guest image boots under the installed Cloud Hypervisor version and can be shut down with reliable process and socket cleanup.

Keep these documents aligned with the implementation. Record actual host versions, commands, results, and limitations rather than inferred or fabricated outcomes.
