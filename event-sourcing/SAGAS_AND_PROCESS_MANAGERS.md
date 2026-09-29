---
title: "Event Sourcing: Sagas and Process Managers"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Sagas and Process Managers

*Sagas and process managers are not the same pattern. Sagas are documents travelling through steps; process managers are stateful coordinators. Choose the simplest thing that can possibly work.*

---

## The distinction

The two patterns solve the same problem — coordinating multiple services
through a long-running business process — but they are different tools,
commonly misnamed.

| | Saga | Process Manager |
|---|---|---|
| Shape | A **document** defining a process, updated and passed to the next step as it completes | A **stateful actor** (a messaging state machine) that runs the process |
| State | Lives in the document, carried along | Lives in the manager, derived from events (when event-sourced) |
| Routing | Routing slips; most commonly HTTP | Handles messages by correlation id |
| Fit | Forward-flow processes | Forking, backtracking, loops |

**Rule of thumb:** prefer a forward-flow process implemented as a saga —
simpler to reason about and implement. When the process needs forks,
backtracking or loops, a process manager handles those more naturally.
Switching later is normal: a small business change often flips which
pattern is right.

---

## Event-sourced process managers

A process manager is a messaging finite-state machine with a lifecycle that
matches the business process it manages; it is responsible for the process
reaching a known end state. The event-sourced version derives its state
from the events it received rather than storing it separately.

The payoff is **versioning over long lifespans**. A process manager can run
for months — a court-document process literally can. With stored state, v2
must understand v1's state format. Event-sourced, v1 and v2 can read the
same events and interpret them differently: state is derived, not shared.

---

## Versioning long-running processes

The general rule: **do not deploy a new version over an old one**. If the
process changes —

```
P :  take order → cook food → calculate payment → take payment
P':  take order → calculate payment → take payment → cook food
```

— what happens to an order at "take payment" when P' deploys? Does the
food get cooked twice? Instead, tie a process instance to the version it
started on. For explicit handover, emit an interrupt event:

```elixir
%OrderInterrupted{order: ..., state: %{cooked: true, paid: false}}
```

The new process version subscribes to `OrderInterrupted`, reads the state
off the event, and takes over where the old one left off. The event carries
the full state so the handover needs nothing else.
