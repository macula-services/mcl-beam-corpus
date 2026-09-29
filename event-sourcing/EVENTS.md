---
title: "Event Sourcing: Events"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Events

*An event records a finished business fact. It is named in the past tense, it is atomic, and once written it never changes.*

---

## What an event is

An event is a small immutable record saying that something happened in
the domain, with the data needed to understand it later. The name is a
past-tense verb phrase in the language of the business:

```elixir
%SeatReserved{seat_id: "B-12", show_id: "2026-10-03-evening", reserved_by: "cust-981"}
%RefundIssued{order_id: "ord-4410", amount_cents: 1_250, reason: :double_charge}
%ShelfRestocked{shelf_id: "aisle-7-3", sku: "tea-green-100g", quantity: 24}
```

Events are the unit every other part of the system builds on:
aggregates fold them, projections turn them into read models, adapters
translate them for other services.

## Rules that keep events useful

### Name completed facts

An event is written after the thing has happened, so the name is in the
past tense. States that last a while are not events; their start and end
are. Instead of `ImportRunning`, record `ImportStarted` and
`ImportFinished`, and let a projection work out that an import is in
progress between the two.

### One event, one atomic fact

Ask what the log would say if the node died halfway through the action.
If the honest answer is "we could not tell whether it happened", the
event is too big. Split it so that each recorded event corresponds to
something that either fully happened or did not happen at all. Greg
Young calls this the power-cable test.

### Never edit, only append

A wrong event is corrected by a later event (`RefundReversed`,
`AddressCorrected`), not by rewriting the stored one. The unedited log is
what makes audit and replay trustworthy. Changes to an event's *shape*
are handled by versioning (new event types, or upcasting old ones on
read), never by rewriting history in place.

### Say what happened, not what changed

`OrderStatusChanged{status: 4}` tells a reader that a field moved.
`OrderDispatched` tells them why. Intent-revealing events can be
reinterpreted later; field-diff events cannot.

## Naming check

| Avoid | Prefer | Reason |
|-------|--------|--------|
| `ImportRunning` | `ImportStarted`, `ImportFinished` | Duration is derived, not recorded |
| `CustomerChanged` | `CustomerRelocated`, `CustomerRenamed` | "Changed" hides the business meaning |
| `ShipOrder` | `OrderShipped` | Imperative names belong to commands |
| `OrderCreated` / `OrderUpdated` | `OrderPlaced` / `OrderAmended` | CRUD verbs describe storage, not the business |

If nobody can say the past tense of an action, it is probably not a fact
yet; it is a request, which makes it a command
(see [COMMANDS_AND_QUERIES](COMMANDS_AND_QUERIES.md)).

## On the BEAM

Events are usually structs (Elixir) or maps/records (Erlang) with a
schema version in the envelope. Keep them plain data: no pids, refs or
funs, because they must survive serialisation and be readable by code
written years later.

## Why it matters

A badly shaped event leaks ambiguity to every consumer, and each of them
ends up inventing its own interpretation. Getting the event right at
design time is far cheaper than living with divergent workarounds.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Greg Young, *Versioning in an Event Sourced System*, Leanpub (last updated 2017; free to read online). https://leanpub.com/esversioning/read
- Martin Fowler, "Event Sourcing", martinfowler.com, 2005 (free). https://martinfowler.com/eaaDev/EventSourcing.html
- Microsoft, "Event Sourcing pattern" (event design and versioning sections), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
