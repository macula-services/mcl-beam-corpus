---
title: "Event Sourcing: Checkpoints"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Checkpoints

*A checkpoint is a pointer into the log: where this consumer got to. Durable where the consumer's progress must survive a restart.*

---

## Why they exist

Consumers follow the log and act on events. When the power goes out:

1. Replaying terabytes of history is not an option — the system must come
   back fast.
2. Actions that must happen once must not happen again on restart.

A checkpoint solves both: remember the position of the last successfully
processed event, resume from there. The simplest form is a byte offset into
the log.

---

## Choosing a checkpoint

Prefer the simplest mechanism that meets the durability need.

| Kind | Survives restart | Cost | Use when |
|------|-------------------|------|----------|
| **Memory** | No — replays from start | Free | Tests; projections cheap to rebuild; anything rebuilt on restart by design |
| **File** | Yes, if implemented durably | Low | Simple services, one instance |
| **Database** | Yes, shared | Medium | Multiple instances must not double-consume |
| **Checkpoint stream** | Yes, in the event store | Store writes | Central visibility + audit; low-to-medium volume |
| **Shared** | Depends on backing | Coordination | Multiple consumers agreeing on one position |
| **Consensus** | Yes | Very high | Majority-verified position; see below |

---

## Database checkpoint — the atomic write

The reason to keep a checkpoint in the database is that **the checkpoint
and the projection data can be written in one transaction**. That removes
the entire class of edge cases around retries: a projection that wrote rows
but lost its checkpoint replays and double-writes (make writes idempotent
anyway); one that saved its checkpoint but lost the write silently skips
events. Atomicity makes both impossible.

Keeping the checkpoint *with* the projection row also enables fine-grained
optimistic concurrency and lets a query return "data as of position N".

---

## Checkpoint stream — central visibility, write amplification

Store the checkpoint as the last event in its own stream: a tail read gives
the position, and the stream *is* the audit of every position the consumer
ever held. You can see where a projection was at 14:01:27, not just where it
is now.

Costs:

- **Write amplification.** Every consumer writes back per batch processed;
  with many read models the store sees 3-5x the domain writes. Pathological
  at high volume.
- **WORM media** accumulates useless checkpoint history forever.

Default to a checkpoint stream on low-to-medium volume systems; it is
simple to move away from later when amplification becomes material.

---

## Shared and consensus checkpoints

- **Shared**: multiple consumers on one checkpoint. Typical use: several
  instances agreeing which is active, or splitting events (odd/even) to
  scale out a consumer. Absolute ordering is lost — a read model may have
  "shipped" before "paid". Worth it when not all projections need order.
- **Consensus**: a majority of subscribers must agree on the position.
  Buy: with geographically distributed read models, a quorum read is
  guaranteed to include a replica that is up to the checkpoint. Cost: if
  a majority is down, no consensus, no progress. **Usually a sledgehammer
  for a fly** — verify you need it before you pay for it.

---

## The monitoring angle

Checkpoints are the system's own observability: each consumer's position
vs the log's head is, on average, a wall-clock estimate of how far behind
that consumer is. A single `SELECT * FROM checkpoints` — or a dashboard
reading checkpoint streams — shows the health of every consumer and its
SLA in one place. This is a free benefit of the log-structured design;
use it before building bespoke monitoring.
