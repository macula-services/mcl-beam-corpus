---
title: "Event Sourcing: Event Logs"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Event Logs

*Where events live. A log is the physical append-only record; a stream is the logical sequence of events for one thing. Choose the simplest log that meets your durability and scale needs.*

---

## Log, stream, and sourcing

- An **event log** is the storage: records appended in order and never
  modified.
- An **event stream** is a named, ordered subset of events, typically
  everything that happened to one aggregate (`reservation-4410`).
- **Event sourcing** is the decision to treat the log as the source of
  truth ([EVENT_SOURCING](EVENT_SOURCING.md)). You can keep a log without
  making that decision, and still benefit from being able to replay it.

## Kinds of log

### A single append-only file

Serialise each event and append it to a file. For a desktop tool, an
embedded device, or a service that holds only a few thousand events and
can replay them at startup, this is a perfectly sound design. Its limits
are size and continuous operation. If you rely on it, treat durability
seriously: detect a torn final record after a crash (length prefix plus
checksum), and `fsync` at the points where you promise the caller the
event is safe.

### A segmented log

The log is split into fixed-size segment files. Only the newest segment
is written; full segments are sealed and never change again. Sealed,
immutable segments are what make the rest easy:

- they can be copied, cached and replicated to readers without
  coordination;
- retention can work per segment: rewrite a sealed segment without the
  records that have expired, then swap it in, and merge small leftovers;
- they suit write-once storage.

Kafka's log segments and the chunk files of KurrentDB (formerly
EventStoreDB) are well-known examples of this layout.

### A replicated or distributed log

The log is replicated across nodes for availability, via a consensus
protocol such as Raft when the nodes trust each other, or via a
Byzantine-tolerant protocol (a blockchain is one) when they do not. You
gain availability and pay in latency and operational complexity. Most
systems need a replicated log among trusted nodes at most, and many need
less. On the BEAM, Raft-based stores built on `ra` (for example Khepri)
are the usual route.

## Streams

Most reads and writes in an event-sourced system are per stream:

| Operation | Role |
|-----------|------|
| Append (with expected version) | The write path, with optimistic concurrency |
| Read forward from a position | Rehydrate an aggregate, rebuild a projection |
| Subscribe | Receive new events as they are appended; how projections, process managers and other services are fed |
| Delete / truncate | Often restricted or absent by design; where erasure is required, prefer keeping personal data out of events |

Streams are the natural partition key: a system may have millions of
small streams, much as a document database has many documents. Many
stores create a stream implicitly on its first append.

Because a store that supports subscriptions pushes new events to
interested parties, it frequently doubles as the messaging backbone
between components.

## Choosing

1. Start with the simplest log that meets the durability requirement. A
   file is a reasonable first system.
2. Move to segments when you need retention, replication to readers, or
   a history too large to rewrite.
3. Move to a replicated log when losing a single node must not stop
   writes. Reach for untrusted-writer designs only when the problem
   really has untrusted writers.

Related: [CHECKPOINTS](CHECKPOINTS.md) for how consumers remember where
they are in a log, [STREAM_FORK_AND_JOIN](STREAM_FORK_AND_JOIN.md) for
deriving new streams from existing ones.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Martin Fowler, "Event Sourcing", martinfowler.com, 2005 (free). https://martinfowler.com/eaaDev/EventSourcing.html
- Microsoft, "Event Sourcing pattern" (event store options), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
