---
title: "Architecture: Event-Driven Architecture"
layer: guide
audience: [agent, human]
stage: stable
---

# Architecture: Event-Driven Architecture

*Events carry the integration between decoupled components. Two topologies decide who owns the workflow: the mediator or the brokers.*

---

## The model

Components are **event processors**: self-contained, independent, highly
decoupled units, each performing a single business task. They do not
call each other — they publish and consume events through **event
channels** (message queues or topics). A processor publishes an event
for what it just did; others react.

The decoupling pays off in evolution: an event can be published and
picked up by nobody — common when adding functionality, and the
system keeps working.

---

## Topology 1: mediator

A central **event mediator** owns the workflow:

```
initial event ──▶ mediator ──▶ processing events ──▶ channels ──▶ processors
                      ▲                                              │
                      └───────────── results come back ─────────────┘
```

- The mediator knows the steps of the process but performs **no
  business logic** — it sends a processing event per step and waits.
- Topics are typical, so one event can fan out to several processors.
- Fit: workflows needing central orchestration, error handling, and
  retries in one place. It is the saga/process-manager shape — see
  [SAGAS_AND_PROCESS_MANAGERS](../event-sourcing/SAGAS_AND_PROCESS_MANAGERS.md).

**Match the mediator to the complexity.** An integration hub doing
sophisticated orchestration is a recipe for failure; a BPM engine doing
simple routing is too. Simple routing → integration hub; complex
processes (human steps) → process manager.

---

## Topology 2: broker

No central orchestrator — flow is a chain through a lightweight
broker:

```
processor A ──▶ broker ──▶ processor B ──▶ broker ──▶ processor C
```

- Each processor does its task and publishes an event stating what it
  did; the next processor picks it up.
- Fit: simple, linear event flows where central orchestration is
  unwanted overhead.

---

## The trade

| | Mediator | Broker |
|---|---|---|
| Workflow ownership | Central, explicit | Distributed, implicit |
| Flow visibility | In the mediator | Reconstructed from the chain |
| Change | One place, one deploy | Per processor |
| Failure handling | Centralised retries | Per processor |
| Coupling | Processors decoupled, mediator knows all | Fully peer-to-peer |

## Choosing

- **Mediator** when the process is a process: ordered steps, error
  paths, retries, human intervention.
- **Broker** when the flow is simple and linear and nobody needs to
  own it.
- In event-sourced systems the topology question lands on the same
  axis as sagas vs process managers: forward-flow processes prefer the
  simpler shape (broker/saga); branching, backtracking processes
  belong to the stateful centre (mediator/process manager).
