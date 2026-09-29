---
title: "Event Sourcing: Domain Modeling"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Domain Modeling

*Before aggregates, events and read models there is the model itself: which concepts exist, what they are called, and where each meaning stops applying.*

---

## The vocabulary

| Term | Meaning |
|------|---------|
| **Domain** | The problem area and what the people working in it already know |
| **Domain model** | The concepts and rules you choose to capture in software, a deliberate simplification |
| **Bounded context** | The boundary within which one model, and one meaning per term, applies |
| **Ubiquitous language** | The shared vocabulary used identically by domain experts and in the code, inside one context |

A bounded context is primarily a *linguistic* boundary. Deployment
units, repositories and teams often line up with it, but the defining
test is: inside this line, does every term have exactly one meaning?

## Language first

Inside a context, the words people use in conversation should be the
words in the code: module names, command names, event names. When a
domain expert says "a seat is held until payment, then allocated", the
model should have `SeatHeld` and `SeatAllocated`, not `SeatStatusChanged`.

When one word carries two meanings, you have found two contexts. A
"ticket" at the box office is something a customer bought; in the
support desk it is a complaint. Forcing both into one model produces a
bloated type that serves neither well.

## Entities and values

| | Entity | Value |
|---|---|---|
| Identity | Has an id that persists through change | None; defined entirely by its attributes |
| Equality | Same id, same thing | Same attributes, same value |
| Change | Evolves over time | Immutable; replaced, not modified |
| Examples | a showing, a customer account | a price in cents with currency, a seat position, a date range |

In event-sourced systems, entities with their own lifecycle and rules
usually become aggregates (identified by stream id), and values become
the fields of commands and events.

## Letting the types carry the rules

A functional style models the domain directly in data types, so that
many invalid situations cannot even be constructed ("make illegal
states unrepresentable", Yaron Minsky's phrase, is a theme Scott
Wlaschin's book develops at length).

```elixir
defmodule Money do
  @enforce_keys [:cents, :currency]
  defstruct [:cents, :currency]

  def new(cents, currency) when is_integer(cents) and cents >= 0 and currency in [:eur, :gbp],
    do: {:ok, %Money{cents: cents, currency: currency}}
  def new(_, _), do: {:error, :invalid_money}
end
```

- Validate at the edge with constructors that return `{:ok, value}` or
  `{:error, reason}`; the core then works only with values known to be
  valid.
- Represent alternatives explicitly (`{:held, until}` vs
  `{:allocated, customer_id}`) instead of a pile of nullable fields.
- Model a business workflow as a pipeline of functions: command in,
  events or a refusal out. That is exactly the shape of an aggregate's
  decide function ([AGGREGATES](AGGREGATES.md)).

Elixir and Erlang check these rules at runtime rather than at compile
time; typespecs, Dialyzer and pattern matching in function heads give
part of the benefit a static type system would.

## Sizing contexts

There is no formula, only trade-offs:

| Approach | Gain | Risk |
|----------|------|------|
| Few large models | One interpretation everywhere | Every change touches many people; the model grows vague |
| Many small contexts | Each evolves independently | Translation between contexts; concepts appear more than once |

A practical heuristic: concepts that must stay consistent at the moment
of a decision belong together; concepts that change for different
reasons, owned by different people, belong apart.

## Contexts in an event-sourced system

- **Events belong to a context.** `OrderPaid` in billing and `OrderPaid`
  in fulfilment may be different facts with different data.
- **Internal events stay internal.** Other contexts receive explicit
  integration events ([INTEGRATION_EVENTS](INTEGRATION_EVENTS.md)).
- **The language names the events.** If the experts have no past-tense
  word for it, it is probably not an event yet ([EVENTS](EVENTS.md)).

## Why it matters

Good boundaries make replay, projections and independent evolution
cheap. Poor ones turn the event log into a shared global variable where
every consumer depends on every producer's internal vocabulary.

## Sources

- Scott Wlaschin, *Domain Modeling Made Functional: Tackle Software Complexity with Domain-Driven Design and F#*, 1st edition, The Pragmatic Programmers, 2018. https://pragprog.com/titles/swdddf/domain-modeling-made-functional/ ; free talk and slides by the author: https://fsharpforfunandprofit.com/ddd/
- Alex Lawrence, *Implementing DDD, CQRS and Event Sourcing*, Leanpub, 2021 (now retired from sale on Leanpub). https://leanpub.com/implementing-ddd-cqrs-and-event-sourcing
- Eric Evans, *Domain-Driven Design Reference*, Domain Language, 2015 (free PDF). https://www.domainlanguage.com/ddd/reference/
- Martin Fowler, "Bounded Context", martinfowler.com, 2014 (free). https://martinfowler.com/bliki/BoundedContext.html
- Martin Fowler, "Value Object", martinfowler.com, 2016 (free). https://martinfowler.com/bliki/ValueObject.html
