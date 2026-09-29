---
title: "Event Sourcing: Aggregates"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Aggregates

*The consistency boundary: the object that decides whether a command becomes events, by folding its own history.*

---

## What an aggregate is

An **aggregate** is the unit that decides: it receives a command,
loads its own event history, and either emits new events or rejects
the command. It is the write side's decision point, and its stream is
its memory.

```
command ──▶ aggregate ──▶ events (appended to its stream)
                ▲
                └── hydration: fold its stream into current state
```

Hydration is a left-fold: `state = fold(apply, initial, events)`.
The aggregate's current state is never stored — it is always derived
by replaying the stream, which is why the aggregate can be rebuilt
anywhere, any time.

---

## The aggregate root

An aggregate has one **root** — the only element reachable from
outside:

- External code references the root only, never the aggregate's inner
  parts.
- The root exposes the command functions; the invariants are enforced
  inside the boundary.
- Sub-entities inside the aggregate may have only **local identity** —
  meaningful within the aggregate, never across it.

The root is what makes the boundary real: every change to the
aggregate passes through one door, so consistency can be checked in
one place. Without the boundary, related changes are scattered and
transactional guarantees become impossible.

---

## The rules

| Rule | Why |
|------|-----|
| One stream per aggregate | The stream *is* the boundary; replay rehydrates the whole |
| Load full history, then decide | A decision made on partial history is a wrong decision |
| Commands may be rejected | The aggregate says no; zero events is a valid outcome |
| Invariants live inside | "An order cannot ship before payment" is checked at the root, not by callers |
| Small aggregates | Big aggregates mean many commands contend on one stream; split them |
| Reference other aggregates by id | Never hold another aggregate's object — ids, then a saga/process manager coordinates (see [SAGAS_AND_PROCESS_MANAGERS](SAGAS_AND_PROCESS_MANAGERS.md)) |

---

## Aggregate design heuristics

- **Name by behaviour, not data.** `Order` decides ordering rules;
  if it only holds fields, it is a DTO, not an aggregate.
- **Every command has one aggregate that owns it.** If no aggregate
  clearly owns the command, the boundary is wrong — or the command
  belongs to a process manager.
- **"Invariant" means across events.** An invariant the events can
  never violate needs no check — only rules spanning multiple events
  justify the aggregate's existence.
- **Version by event count.** Concurrency control = append with the
  expected stream position; a mismatch is a conflict to retry, never
  a silent overwrite.

---

## Why it matters

The aggregate is where "correct" lives in an event-sourced system:
the fold is pure, the invariants are explicit, and everything else —
projections, queries, integrations — reads what the aggregates
decided. Get the aggregate boundaries right and the rest of the system
is derivation; get them wrong and every consumer inherits the
ambiguity.
