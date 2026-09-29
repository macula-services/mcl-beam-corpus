---
title: "Event Sourcing: CQRS"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: CQRS

*Command Query Responsibility Segregation: give changing the system and asking the system separate models. The idea is small; the consequences are what need care.*

---

## What it is

CQRS takes the command/query distinction
([COMMANDS_AND_QUERIES](COMMANDS_AND_QUERIES.md)) and applies it to the
structure of a component. Instead of one model that both enforces
business rules and serves every screen, there are two:

- a **write model** that accepts commands, enforces invariants and
  records the outcome;
- one or more **read models** shaped for the questions people ask,
  denormalised and free of business rules.

The smallest version is two modules where there used to be one:

```elixir
defmodule Box.Office do          # write side: commands only
  def reserve_seat(show_id, seat_id, customer_id), do: …
  def release_seat(show_id, seat_id), do: …
end

defmodule Box.Office.Views do    # read side: queries only
  def seats_available(show_id), do: …
  def reservations_for(customer_id), do: …
end
```

Everything beyond that (separate stores, separate deployments, events
flowing between the sides) is an implementation choice layered on top.

## Common misreadings

- **"CQRS means microservices."** No. Two modules in one OTP application
  already qualify. Splitting into separately deployed services is a
  scaling or ownership decision.
- **"CQRS requires event sourcing."** No. A write model over ordinary
  tables can feed read models too. Event sourcing makes it convenient,
  because the events are exactly what the read side needs to subscribe to
  ([PROJECTIONS](PROJECTIONS.md)).
- **"CQRS is complicated."** The pattern is not. The complexity people
  report usually comes from what they adopted alongside it: messaging,
  eventual consistency, multiple databases.

## What it buys

- Each side is optimised for its own job: the write side for correctness
  and contention, the read side for query speed and shape.
- The read side can scale out independently and can have as many models
  as there are distinct questions.
- Rules live in one place (the write model); screens do not accumulate
  business logic.

## What it costs

- With separate stores, read models lag behind writes. The UI and API
  must be designed for eventual consistency (for example, return the new
  stream version from a command and let a client wait until a read model
  has reached it).
- More code paths to test, deploy and monitor.

## When to reach for it

- Reads and writes have clearly different shapes or loads.
- Several very different views of the same data are needed.
- The domain has real rules worth guarding in a focused write model.

For simple CRUD screens over simple data, a single model is fine.

## Sources

- Greg Young, *CQRS Documents*, self-published PDF, 2010 (free). https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf
- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Martin Fowler, "CQRS", martinfowler.com, 2011 (free). https://martinfowler.com/bliki/CQRS.html
- Microsoft patterns & practices, *Exploring CQRS and Event Sourcing* (the CQRS Journey), 2012 (free online). https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10)
- Microsoft, "CQRS pattern", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
