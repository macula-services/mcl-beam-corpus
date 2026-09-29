---
title: "Architecture: Event-Driven Architecture"
layer: guide
audience: [agent, human]
stage: stable
---

# Architecture: Event-Driven Architecture

*Components cooperate by announcing what happened instead of calling each other. The main design decision is whether a central coordinator owns the workflow (mediator) or the workflow emerges from components reacting to each other (broker).*

---

## The model

An event-driven system is made of **event processors**: components that
each do one job, consume events from **channels** (queues, topics, log
streams) and publish events about what they did. A publisher does not
know who, if anyone, is listening.

That indifference is the main benefit. A new consumer can be added
without touching the producer, and an event nobody consumes yet costs
little. The main costs are the flip side: no single place shows the
whole flow, delivery is usually at least once, and the system is
eventually consistent.

"Event-driven" also covers several different intentions (Martin Fowler
separates them): notifying that something happened, carrying the new
state so consumers need not call back, event sourcing, and CQRS. Know
which one you mean before arguing about topology.

## Mediator topology

A **mediator** receives an initiating event and drives the steps of the
workflow, sending a message to each processor in turn and reacting to
the results:

```
OrderPlaced ──▶ mediator ──▶ ReserveStock  ──▶ stock processor
                   ▲  │                              │
                   │  └──▶ ChargeCard ──▶ payment processor
                   └──────── results (StockReserved, CardCharged, …)
```

- The mediator knows the order of steps and handles failures, retries
  and timeouts in one place. It should not contain the business rules of
  the steps themselves.
- Fit: workflows with ordering, error paths, compensation or human
  steps.
- Risk: the mediator is a point of coupling and can become a bottleneck
  or a single point of failure.
- Size the mediator to the job. A lightweight router is enough for
  simple routing; a full workflow or BPM engine is justified only for
  long, complex processes. Using either for the other's job hurts.

In event-sourced systems the mediator is a process manager
([SAGAS_AND_PROCESS_MANAGERS](../event-sourcing/SAGAS_AND_PROCESS_MANAGERS.md)).

## Broker topology

No coordinator. Each processor reacts to events it cares about and
publishes its own:

```
OrderPlaced ──▶ stock processor ──▶ StockReserved ──▶ payment processor ──▶ CardCharged ──▶ …
```

- Fit: simple, mostly linear flows, and fan-out where many independent
  consumers react to the same event.
- Risk: the workflow exists only implicitly, spread over subscriptions.
  Error handling is local to each processor, and a multi-step
  transaction has no owner that can restart or compensate it.

The broker style is often called choreography and the mediator style
orchestration.

## Comparison

| | Mediator | Broker |
|---|---|---|
| Who owns the workflow | The mediator, explicitly | Nobody; it emerges |
| Seeing the whole flow | Read the mediator | Reconstruct from traces and correlation ids |
| Changing the flow | One component | Possibly several processors |
| Failure handling | Central retries and compensation | Per processor |
| Coupling | Processors decoupled from each other, mediator knows all | Processors know only event types |

## On the BEAM

Within one node, `Registry`-based or `:pg` process groups give cheap
publish/subscribe between processes; between nodes, distributed Erlang
or an external broker carries the events. A mediator is naturally a
supervised `gen_statem` or GenServer per workflow instance. Whatever
the transport, give every event correlation and causation ids
([IDS_AND_CORRELATION](../event-sourcing/IDS_AND_CORRELATION.md)) so
broker-style flows can be traced.

## Choosing

- **Mediator** when the process is a real process: ordered steps, error
  paths, compensation, waiting, people.
- **Broker** when the flow is short and linear, or when consumers are
  genuinely independent.
- This is the same axis as saga versus process manager: forward-only
  flows suit the lighter shape, branching and long-running ones the
  stateful coordinator.

## Sources

- Mark Richards, *Software Architecture Patterns*, 1st edition, O'Reilly Media, 2015 (https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/); 2nd edition, O'Reilly Media, 2022 (https://www.oreilly.com/library/view/software-architecture-patterns/9781098134280/)
- Martin Fowler, "What do you mean by 'Event-Driven'?", martinfowler.com, 2017 (free). https://martinfowler.com/articles/201701-event-driven.html
- Microsoft, "Event-driven architecture style" (broker and mediator topologies), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven
