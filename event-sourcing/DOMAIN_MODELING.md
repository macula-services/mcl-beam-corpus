---
title: "Event Sourcing: Domain Modeling"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Domain Modeling

*Bounded contexts, entities, value objects, ubiquitous language. The modeling layer that decides what the aggregates, events and read models are about.*

---

## The three terms, untangled

| Term | What it is |
|------|------------|
| **Domain** | The existing knowledge around a problem — what people already know |
| **Domain model** | The abstractions *you* build, in a custom language, from that knowledge |
| **Bounded context** | The boundary you define, inside which one model's terms have one meaning |

A bounded context is "an explicit boundary within which a domain model
exists". The point is not architecture — it is **applicability**: inside
the boundary, a term means exactly one thing, so the ubiquitous
language needs no translation.

---

## The ubiquitous language

Within a context, everyone — domain experts and code — uses the same
words for the same things. `Order`, `Fulfillment`, `Payment` mean the
same in a meeting and in a module name. The payoff:

- No translation between business and code; the code *is* the
  language.
- A term that means two things signals **two contexts** — the classic
  example: `Customer` in the billing context vs `Customer` in the
  support context are different models.

---

## Entities and value objects

| | Entity | Value object |
|---|---|---|
| Identity | Persistent, unique id | None — two equal values are interchangeable |
| Equality | By id | By value (all fields) |
| Mutability | Changes over its lifetime | Immutable |
| Examples | `Order`, `Account`, `User` | `Address`, `Money`, `TimeRange` |

The test: "Chris has the **same name as**..." — value object. "This
**is** the order I placed" — entity. In event-sourced systems this
maps naturally: entities are aggregates (identified by stream id);
value objects are the fields inside events and commands.

---

## Model and context sizing

There are no strict rules; there are risks:

| Approach | Promise | Risk |
|----------|---------|------|
| One large model per domain | Uniform interpretation, no ambiguity | Complexity grows with size; every change must reach every stakeholder |
| Smaller models per context | Each context evolves alone | Translation between contexts; duplicated concepts |

Rule of thumb: **split by how terms change independently.** Two
concepts that change for different reasons belong in different
contexts; concepts that must stay consistent with each other belong in
one — which is also the aggregate rule (see [AGGREGATES](AGGREGATES.md)).

---

## Contexts and the event-sourced architecture

The bounded context is the natural unit of the write side:

- **One context, one event store namespace.** Events are facts *of a
  context*; `OrderPaid` in billing may be a different fact than
  `OrderPaid` in fulfilment.
- **Contexts integrate by integration events**, not by sharing models
  — see [INTEGRATION_EVENTS](INTEGRATION_EVENTS.md).
- **A context's internal events are private.** Only the adapter's
  denormalized contracts cross the boundary.
- **The ubiquitous language names the events.** If the business cannot
  say the past tense of an action, the event does not exist yet — see
  [EVENTS](EVENTS.md).

## Why it matters

Modeling is the layer that decides whether the event-sourced machinery
pays off. Good boundaries make replay, projections, and independent
evolution easy; bad boundaries make the event log a shared global
variable — every consumer coupled to every producer's internal
vocabulary.
