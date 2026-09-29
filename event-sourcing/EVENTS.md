---
title: "Event Sourcing: Events"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Events

*An event is a fact which has occurred at a point in time. Past tense. Atomic. Immutable.*

---

## The definition

An **event** is a piece of data representing behaviour that has completed.
It is named as a verb in the past tense:

```
TaxiFarePaid     { id="73648423", ride="27394572", time="2022-04-23T18:25:43.511Z" }
ItemMoved        { id="24728347", from="17", to="28" }
ItemDeactivated  { id="37237787", reason="no longer available" }
```

---

## The rules

### 1. Past tense, always

The action has completed when the event is created. Never name an event
for something in progress: `XYZProcessRunning` is wrong; you want
`XYZProcessStarted` and `XYZProcessCompleted`. The "running" span is
*inferred* as the time between them — and later events can refine the
interpretation without renaming anything.

### 2. Atomicity — the power cable test

Every event is an atomic action that has completed. For any event you
model, ask: **what if somebody pulls the power cable?** If the answer
is "we do not know whether it happened", the action is not atomic — split
it into started/completed events so the system's state is always derivable
from what is actually recorded.

### 3. Immutable, never edited

Events are appended, never rewritten. "Correcting" history means appending
a compensating or correcting event, not altering the old one. The log is
the audit trail; editing it destroys the property the whole system rests on.

---

## Naming discipline

| Wrong | Right | Why |
|-------|-------|-----|
| `ProcessRunning` | `ProcessStarted` + `ProcessCompleted` | Ongoing vs completed |
| `ItemStateChanged` | `ItemMoved` | "Changed" hides what happened |
| `UpdateOrder` | `OrderPaid`, `OrderShipped` | Command names on events |

Events describe what *did* happen. If you find yourself unable to name the
past tense of the thing, the thing probably is not a fact yet — it is a
command in disguise.

## Why this matters

Events are the atoms the whole architecture compounds from: aggregates
fold them, projections interpret them, integration consumers translate
them. One event that breaks the past-tense/atomicity rules propagates
ambiguity through every downstream consumer, and each consumer ends up
re-deriving the same clarifications independently.
