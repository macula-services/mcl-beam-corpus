---
title: Event Sourcing & CQRS
layer: index
audience: [agent, human]
stage: stable
---

# Event Sourcing & CQRS

*The write model, the read models, and everything between.*

These notes describe patterns, language-neutral where possible. They are
the knowledge behind CMD/PRJ/QRY-style architectures: commands decide,
events record, projections answer.

---

## Notes

| Note | Covers |
|------|--------|
| [PROJECTIONS](PROJECTIONS.md) | Deriving read models from events: simple, updating, batched |

## Planned

- Event logs: one stream, segmented streams, distributed logs
- Checkpoints: memory, file, database, consensus
- Sagas & process managers: long-running workflows, compensation
- Ids & correlation: message, causation, correlation, conversation ids
- Stream fork & join
- Optimistic concurrency

## The landscape at a glance

```
Command ──▶ Aggregate ──▶ Event(s) ──▶ Event store (append-only)
                                        │
                                        ▼
                              Projection (per read model)
                                        │
                                        ▼
                              Query side: read models
```

- **Commands** request; they may be rejected and produce zero events.
- **Aggregates** decide: load their own history, validate, emit events.
- **Events** record, in the past tense; they are never edited.
- **Projections** derive read models from events, tracking progress with
  a **checkpoint**.
- **Queries** read the read models; they never touch the event store.

The event store is the single source of truth. Read models are disposable
artifacts: delete one, replay the events, rebuild it.
