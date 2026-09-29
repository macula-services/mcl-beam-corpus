---
title: "Event Sourcing: Event Logs"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Event Logs

*An event log is every event written to a log. An event stream is an ordered set of events. Event Sourcing is making the log the source of truth — a separate decision.*

---

## Log vs sourcing — do not conflate them

- **Event Log**: every event the system produces is written to a log.
- **Event Sourcing**: the log is *also* the primary source of truth for the
  entire system.

There can very well be an event log without event sourcing — market-data
systems have logged every tick for decades and replay the day's file to run
new analyses — and that alone delivers value.

---

## Implementations, from simple to heavy

### Appending file event log

Literally append events to a file. Not a joke: with hundreds to low
thousands of events, a JSON file replays in under a second. Correct where
the log is client-side, or the system has a natural "off" period (a trading
system that runs 8 hours and closes the day). It fails at 24/7 scale.

Durability caveats for when events matter:
- Handle partial writes (append, then write a checkpoint confirming the
  write completed).
- Disks lie about durability; test what you trust.

### Segmented event log

The log is a series of fixed-size segments (EventStore defaults to 256 MB),
filled one at a time. Once a segment is filled it becomes **immutable** —
the property everything else builds on:

- Write-once media works.
- Immutable segments replicate trivially to many readers, geographically.
- **Scavenging**: copy a segment, dropping expired data (e.g. records
  older than two weeks), then swap it in. Old chunks losing 90% of data
  get merged into larger files.

### Distributed event log

The log replicated across machines for availability, where the writers are
not fully trusted — the blockchain is the famous instance. Highly available
but with a significant complexity cost. **Gate this hard**: most systems
never need it.

---

## Event streams

An **event stream** is an ordered set of events. Events are dealt with as
members of a stream, not in isolation. All events for one aggregate live in
one stream; hydrating the aggregate is replaying its stream.

A stream is the primary partition point: typical systems have hundreds of
thousands of streams. In an event-sourced system a stream is what a
"document" is in a document database.

The operations a stream supports, and what they enable:

| Operation | Note |
|-----------|------|
| Create | Some stores auto-create on append (EventStore does) |
| Append | The write path |
| Read | Replay for hydration or rebuilding |
| **Subscribe** | The defining operation: read models, other services, clients — all consume via subscription |
| Delete | Often absent on purpose: no delete is a compliance *feature* |

A subscribe-capable store doubles as a message dispatcher, which is how
most event-sourced systems move data between services.

## Choosing

Start with the simplest log that works; the appending file is a completely
reasonable first system. Move to segmented when immutability, scavenging
or scaling matter. Reach for a distributed log only when availability with
untrusted writers is a real requirement.
