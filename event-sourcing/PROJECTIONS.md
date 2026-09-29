---
title: "Event Sourcing: Projections"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Projections

*Deriving read models from events. The event store is the source of truth; read models are disposable artifacts.*

---

## The pattern

A **projection** consumes events from the event store and upserts a read
model optimized for queries. It is the "write" side of the query service
in CQRS.

```
Event store
    │  subscription (per projection)
    ▼
Projection
    │  INSERT / UPDATE
    ▼
Read model (table, report, file)
    │  SELECT
    ▼
Query handler
```

**Principles**

- Never query external systems in a projection: every value the read
  model needs must be in the events, or derived from them.
- A read model is disposable. Delete it, replay from the beginning, and
  it is rebuilt identically.
- A projection tracks its position with a **checkpoint** so it resumes
  where it stopped, not from the start.

---

## The three basic kinds

### 1. Simple projection — one row per event

Append a row for every event; never update. Suited to event counts,
audit trails, per-event reports.

| event_id | stream | type | at |
|----------|--------|------|----|
| 412 | order-77 | order_placed | 12:01 |
| 413 | order-77 | order_paid | 12:05 |

### 2. Updating projection — one row per entity

The read model reflects the *current* state of an entity: each event
updates the existing row. Suited to entity views: "order 77 as of now".

### 3. Inserting-update projection — upsert

Update the row if the entity exists, insert it if it does not. The safe
default for entity read models when event order is not guaranteed to
start with a "created" event.

**The inserting variant — insert every state.** Instead of updating one
row, insert a new row per change; everything becomes an insert. Space
unfriendly, but:

- Updates become bulk-insertable — a dramatic speedup on replay.
- The read model keeps every historical state, enabling **as-of / as-at
  queries**: the balance *as of* a moment, or *as at* a moment excluding
  what does not apply yet (a cheque deposited today but settling
  tomorrow). Common in financial systems, and the main reason to pick
  this variant. Pair with scavenging of old rows to bound growth.

---

## Beyond one row

| Kind | What it does | Used for |
|------|--------------|----------|
| **Batched projection** | Collects events and applies them in one batch write (per N events or per time window) | High-volume event streams where per-event writes are too slow |
| **Report projection** | Aggregates across many streams into a summary (counts, totals, groups) | Dashboards, "top N" lists, analytics |
| **File projection** | Writes the read model to a file instead of a database | Exports, artifacts consumed by other systems |

---

## Checkpoints

A projection restarts from its **checkpoint**: the last position it
consumed. The choice of checkpoint storage is a trade:

| Storage | Survives restart | Cost | Use when |
|---------|-------------------|------|----------|
| Memory | No — replays from the start | Free | Development, or projections cheap to replay |
| File | Yes, per node | Low | Simple services, one instance |
| Database | Yes, shared | Medium | Several instances must not double-consume |

Store the checkpoint *with* the write it guards, transactionally when
possible: a projection that wrote rows but lost its checkpoint replays
and double-writes (make writes idempotent to be safe), and one that
saved its checkpoint but lost the write silently skips events.
The full catalogue is in [CHECKPOINTS](CHECKPOINTS.md).

## Batched replay, live tail

A projection often has two phases with different optimisations:

- **Replay (history)**: goal is catching up fast. Batch heavily — write a
  CSV and bulk-insert it; an 80-million-row replay is orders of magnitude
  faster that way. Latency to the read model is irrelevant.
- **Live (caught up)**: goal is freshness. Switch to per-event writes.

If both phases share code, switching is a batch-size change: fall behind,
batch up; caught up, individual writes.

---

## Naming

- Projection modules are named for the read model they build:
  `OrderSummaryProjection`, not `OrderProjectionWorker`.
- Projection functions are named for the event they apply:
  `apply_order_placed/2`, `apply_order_paid/2`.
- The write side names the past tense of what happened; never reuse
  command names on the read side.

## Anti-patterns

- **Editing read models by hand.** They derive from events only; a manual
  fix disappears on the next replay.
- **Querying the event store for live queries.** The event store is
  append-only history; queries answer from read models.
- **Sharing a read model between projections.** One projection owns one
  read model; a second consumer needs its own.
