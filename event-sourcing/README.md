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
| [EVENT_SOURCING](EVENT_SOURCING.md) | The pattern itself: state derived from events, replay, reinterpretation, trade-offs |
| [EVENTS](EVENTS.md) | The atom: past tense, atomicity (the power-cable test), immutability |
| [COMMANDS_AND_QUERIES](COMMANDS_AND_QUERIES.md) | Commands return status only; queries never mutate; versioning |
| [CQRS](CQRS.md) | The one-line split; what it is not; when to adopt |
| [EVENT_LOGS](EVENT_LOGS.md) | Appending, segmented, distributed logs; event streams; log vs sourcing |
| [PROJECTIONS](PROJECTIONS.md) | Simple, updating, inserting, batched; checkpoints; replay vs live |
| [CHECKPOINTS](CHECKPOINTS.md) | Memory, file, database, stream, shared, consensus; the monitoring angle |
| [SAGAS_AND_PROCESS_MANAGERS](SAGAS_AND_PROCESS_MANAGERS.md) | The distinction; event-sourced managers; versioning long processes |
| [AGGREGATES](AGGREGATES.md) | The consistency boundary: hydration, aggregate root, invariants |
| [DOMAIN_MODELING](DOMAIN_MODELING.md) | Bounded contexts, entities vs value objects, ubiquitous language |
| [IDS_AND_CORRELATION](IDS_AND_CORRELATION.md) | Message, causation, correlation, conversation ids; the graphs |
| [INTEGRATION_EVENTS](INTEGRATION_EVENTS.md) | Denormalized contracts; aggregators; direct integration's ceiling |
| [STREAM_FORK_AND_JOIN](STREAM_FORK_AND_JOIN.md) | Reindexing with link events; the join's ordering problem |

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
