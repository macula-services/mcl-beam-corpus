---
title: Architecture
layer: index
audience: [agent, human]
stage: stable
---

# Architecture

*The system-level patterns above the BEAM: how services are shaped and connected.*

---

## Notes

| Note | Covers |
|------|--------|
| [EVENT_DRIVEN_ARCHITECTURE](EVENT_DRIVEN_ARCHITECTURE.md) | Mediator vs broker topology, when each fits |
| [ARCHITECTURE_PATTERNS](ARCHITECTURE_PATTERNS.md) | Layered, microkernel, microservices, space-based — which one when |

## One-page summary

Below the BEAM's process model sit system-level decisions: how the
services are layered, how they talk, how they scale. The one that
matters most here is **event-driven architecture** — it is the
organising principle the event-sourcing domain connects to: events
carry integration, and the topology choice (mediator or broker) decides
who owns the workflow.
