---
title: "Event Sourcing: Projections"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Projections

*A projection turns events into a read model shaped for one set of questions. The events are the truth; every read model is a rebuildable by-product.*

---

## What a projection is

A projection subscribes to events and maintains a read model: a table,
an ETS table, a search index, a file. Queries are answered from that
read model, never by folding the event store on the request path
([COMMANDS_AND_QUERIES](COMMANDS_AND_QUERIES.md)).

```
event store ──subscription──▶ projection ──writes──▶ read model ◀──reads── query
```

Three properties to protect:

- **Self-contained.** Everything the read model needs comes from the
  events it consumes (or from other read models it owns). A projection
  that calls a remote service during replay gets different answers each
  time and cannot be rebuilt faithfully.
- **Rebuildable.** Drop the read model, reset the checkpoint, replay, and
  you get the same result.
- **Resumable.** It stores a checkpoint, so a restart continues from the
  last handled event ([CHECKPOINTS](CHECKPOINTS.md)).

## Shapes of read model

| Shape | Write pattern | Good for |
|-------|---------------|----------|
| **Append-only** | One new row per event, never updated | Activity feeds, audit views, counting |
| **Current state** | One row per entity, updated in place (upsert) | "Show me reservation 4410 now" |
| **Versioned rows** | A new row per change, each with valid-from / recorded-at | History and point-in-time queries |
| **Summary** | Aggregates across many streams (counts, totals, top-N) | Dashboards, reports |
| **File or export** | Writes a document, CSV or index instead of rows | Hand-offs to other systems, static sites |

Notes on the choices:

- Prefer **upsert** to separate insert/update paths for current-state
  models. It tolerates a missing "first" event, redelivery, and replays
  that start mid-history.
- **Versioned rows** cost space but make every write an insert, which
  bulk-loads very fast, and they answer two different time questions:
  *as it was known then* (recorded time) and *as it applied then*
  (effective time). Financial systems often need both. Add a retention
  policy so the table does not grow without bound.

## Replay versus live

A projection has two phases with opposite priorities.

- **Catching up** (rebuild, new projection, long outage): throughput is
  everything and freshness does not matter. Batch aggressively: buffer
  thousands of events and write them with a bulk insert or a `COPY`, one
  transaction and one checkpoint per batch.
- **Live** (at the head of the log): freshness matters. Write per event,
  or in small time-bounded batches.

If both phases share one code path parameterised by batch size, the
projection can switch automatically: large batches while far behind,
single events once caught up.

## On the BEAM

A projection is typically one supervised process per read model,
subscribed to the store. For in-memory read models an ETS table owned by
that process (or by a heir) gives concurrent reads without going
through the process; for durable ones, SQLite or PostgreSQL via Ecto,
with the checkpoint in the same database and written in the same
transaction. See [ETS](../beam/ETS.md).

## Naming

- Name the module after the read model it maintains, and the handler
  clauses after the events they apply.
- The read side uses the same past-tense event names as the write side;
  command names never appear in projections.

## Pitfalls

- **Hand-editing a read model.** The next rebuild erases the fix. Fix the
  projection or append a correcting event.
- **Serving queries from the event store.** It is optimised for
  appending and streaming, not for ad hoc reads.
- **Two projections writing one table.** Each read model has exactly one
  owner; a second consumer builds its own.
- **Non-idempotent handlers.** Delivery is usually at least once; a
  handler that increments a counter without checking the event position
  will drift.

Testing projections: see
[TESTING_EVENT_SOURCING](../testing/TESTING_EVENT_SOURCING.md).

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Microsoft, "Materialized View pattern", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view
- Martin Fowler, "Bitemporal History", martinfowler.com, 2021 (free; the two time axes behind versioned read models). https://martinfowler.com/articles/bitemporal-history.html
- Microsoft, "CQRS pattern" (read models built from events), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
