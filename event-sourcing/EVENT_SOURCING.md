---
title: "Event Sourcing: The Pattern"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: The Pattern

*Event Sourcing says that all current state is derived from stored events. That is the entirety of it — trivial, until it is not.*

---

## The definition

**Event Sourcing:** every piece of state in the system is derived from a log of
events. No state exists that did not come from an event.

Equivalently: *current state is a left-fold of previous behaviours* —
`fold(f, state0, events)`.

Two consequences follow from the definition alone:

1. **All state is transient.** A domain object in memory, a table in a
   database — all of it can be thrown away at any moment and rebuilt by
   replaying the same events.
2. **Interpretation can change.** State is an *interpretation* of the event
   log. You can redefine the transformation later, or add a new one, and
   apply it to history. Data you grouped by customer today can be
   regrouped by organisation tomorrow by building a new read model — no
   migration of the old one, which keeps working.

---

## What it is not

- **Not the same as keeping an event log.** An event log records events;
  Event Sourcing *additionally* makes that log the primary source of
  truth for the entire system. You can keep an event log without event
  sourcing — most trading systems log market data — and get replay value
  from it.
- **Not new.** Systems built this way were common until the 1990s, when
  databases started doing it internally. The pattern predates its name.

---

## Events

An **event** is a fact that occurred at a point in time, named in the past
tense. *Alice accepted package AC-378495 at 07:55:42.*

Three rules:

1. **The action has completed.** Events never describe something in
   progress. `BatchJobRunning` is wrong; `BatchJobStarted` and
   `BatchJobCompleted` are right. The "running" span is *derived* from the
   two events.
2. **Every event is atomic.** If an action is not atomic, split it into
   started/completed (and intermediate) events. Test every candidate event
   with: *what if somebody pulls the power cable?*
3. **Past tense, always.** A single violation of this rule adds exponential
   complexity to reasoning about the system.

See [EVENTS](EVENTS.md).

---

## Why people use it

| Property | What it buys |
|----------|--------------|
| Replay | Rebuild any state from scratch; verify; recover |
| Reinterpretation | New read models over old history; answer questions you did not anticipate |
| Audit | The log *is* what happened; no separate audit trail to drift |
| Debuggability | "What was the state at 14:01:27?" is answerable for any point in time |

## The trade-offs

| Cost | Where it bites |
|------|----------------|
| Complexity | Two sides (write/read) where one sufficed; eventual consistency between them |
| Storage | Events accumulate; scavenging/compaction is its own problem |
| Tooling | Few databases are append-only logs; you adopt the log's constraints |

Event Sourcing is a **pattern choice**, not a default. Use it where the
audit, replay and reinterpretation properties are worth the cost.
