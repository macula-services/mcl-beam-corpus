---
title: "Event Sourcing: The Pattern"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: The Pattern

*Store what happened, not what is. Every piece of current state is computed from an append-only record of facts.*

---

## What it is

In an event-sourced system the record of truth is a sequence of events:
facts about things that have already happened. Current state is not
stored as the primary data; it is calculated by applying those events
in order, starting from an empty initial state.

In functional terms, state is a left fold over history:

```elixir
state = Enum.reduce(events, initial_state, &apply_event/2)
```

Everything else follows from taking that sentence seriously.

- **Derived state is disposable.** An in-memory aggregate, an ETS table,
  a SQL read model: any of them can be dropped and recomputed from the
  events.
- **The meaning of history can grow.** Because state is a *calculation*,
  you can write a new calculation later and run it over the old events.
  A report that nobody thought of when the events were written can still
  be produced, without migrating anything that already works.

## How it works

1. A command arrives and is routed to the thing that owns the decision
   (usually an aggregate, see [AGGREGATES](AGGREGATES.md)).
2. The owner rebuilds its state by folding its own event stream.
3. It decides: reject the command, or produce one or more new events.
4. The new events are appended to the stream, guarded by an expected
   version so that two concurrent writers cannot both succeed.
5. Subscribers (projections, process managers, integration adapters)
   pick the events up and derive whatever they need
   ([PROJECTIONS](PROJECTIONS.md), [SAGAS_AND_PROCESS_MANAGERS](SAGAS_AND_PROCESS_MANAGERS.md)).

What an event must look like is covered in [EVENTS](EVENTS.md); where
events are kept in [EVENT_LOGS](EVENT_LOGS.md).

## On the BEAM

The fold maps naturally onto OTP. A process per aggregate instance
(a GenServer, found through a Registry) loads its stream when started,
keeps the folded state in memory, and handles commands one at a time,
which serialises decisions per aggregate without locks. If it crashes,
its supervisor restarts it and it folds its stream again: the event
store, not the process heap, is what must survive. See
[GENSERVER](../beam/GENSERVER.md), [REGISTRY](../beam/REGISTRY.md) and
[SUPERVISION_TREES](../beam/SUPERVISION_TREES.md).

## Keeping a log is not the same thing

Many systems write every event to a log for analysis or replay while
their real state still lives in mutable tables. That is useful, but it
is not event sourcing. The pattern starts when the log becomes the
authoritative source and every other store is derived from it.

## What you gain

| Property | What it gives you |
|----------|-------------------|
| Rebuild | Any derived store can be recreated from scratch |
| New questions over old data | Add a read model later and backfill it from history |
| Audit | The log is the history; there is no separate trail to drift out of sync |
| Time travel | State "as it was at time T" is a fold over a prefix of the log |

## What it costs

| Cost | Where it shows up |
|------|-------------------|
| More moving parts | A write side, a read side, and eventual consistency between them |
| Schema evolution | Old events are never rewritten, so every version must stay readable |
| Growing storage | History accumulates; retention and compaction need a policy |
| Personal data | Immutable events clash with erasure requirements; keep personal data outside events or encrypt it per subject |
| Learning curve | Testing, debugging and operations all change shape |

In Macula codebases event sourcing is the default for significant
business processes. It is still decided per bounded context: where
audit, replay and reinterpretation are not worth the extra machinery,
plain state storage is the better tool.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Greg Young, *CQRS Documents*, self-published PDF, 2010 (free). https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf
- Martin Fowler, "Event Sourcing", martinfowler.com, 2005 (free). https://martinfowler.com/eaaDev/EventSourcing.html
- Microsoft, "Event Sourcing pattern", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
