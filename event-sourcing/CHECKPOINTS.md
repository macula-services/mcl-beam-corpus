---
title: "Event Sourcing: Checkpoints"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Checkpoints

*A checkpoint is a consumer's bookmark in the log: the position of the last event it fully handled. Where you keep it decides what a restart costs and what can go wrong.*

---

## What a checkpoint is for

Anything that follows the log (a projection, a process manager, an
integration adapter) needs to know where to resume after a restart. A
checkpoint records the position of the last event the consumer finished
with: a global log position, a stream version, or a byte offset.

Without one, a restarted consumer must either replay everything (slow,
and dangerous if handling causes side effects) or start at the head
(silently missing events).

## Where to keep it

| Storage | Survives a restart | Cost | Good fit |
|---------|--------------------|------|----------|
| Process memory | No, replays from the start | None | Tests; read models that are cheap to rebuild and rebuilt on every boot by design |
| Local file | Yes, on that node | Low | Single-instance consumers |
| Same database as the read model | Yes | Low to medium | Most projections; see below |
| Its own stream in the event store | Yes | One append per checkpoint | Central visibility of all consumers; modest volumes |
| Shared among several consumers | Depends on backing store | Coordination | Active/standby pairs, or partitioning one consumer's work |
| Agreed by a quorum | Yes | High, and stalls without a majority | Rare; geographically replicated read models that must prove freshness |

## Keep the checkpoint with the write it guards

The strongest option for a projection is to store its checkpoint in the
same database as its read model and update both in **one transaction**.
Then the two can never disagree:

- if the transaction commits, both the rows and the position moved;
- if it fails, neither did, and the event is handled again.

Split across two stores, a crash between the writes leaves either rows
without a moved checkpoint (the event is applied twice on restart) or a
moved checkpoint without rows (the event is silently skipped). The
second failure is the dangerous one. Even with transactions, make
handlers idempotent: delivery is usually at least once.

```elixir
Repo.transaction(fn ->
  apply_to_read_model(event)
  Repo.update_all(from(c in Checkpoint, where: c.name == "seat_map"),
                  set: [position: event.position])
end)
```

Storing a position per row also lets a query report "this answer
reflects the log up to position N".

## Checkpoints as a stream

Appending each new position to a dedicated stream makes every
consumer's progress, and its whole history of progress, visible from the
store itself. The cost is extra writes: each consumer adds its own
appends, so with many consumers checkpoint traffic can rival or exceed
the domain traffic. Batch checkpoint writes (every N events or every
few hundred milliseconds) and switch strategy if the volume becomes
material.

## Sharing and quorum

Several instances can share one checkpoint so that only one is active,
or so they split the events between them. Splitting gives up ordering
across the split: one instance may process "order shipped" before
another has processed "order paid". That is acceptable only for read
models that do not depend on order.

Requiring a majority of replicas to agree on a position is occasionally
justified, and usually far more machinery than the problem needs.

## Monitoring for free

Each consumer's checkpoint compared with the head of the log is its lag.
A single query over the checkpoint table (or a view over checkpoint
streams) shows the health of every projection and process manager at
once, before you build any bespoke monitoring.

See also [PROJECTIONS](PROJECTIONS.md) and, for testing restarts,
[TESTING_EVENT_SOURCING](../testing/TESTING_EVENT_SOURCING.md).

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Microsoft, "Event Sourcing pattern" (idempotency requirements), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
- Microsoft, "Idempotent Consumer pattern", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/idempotent-consumer
