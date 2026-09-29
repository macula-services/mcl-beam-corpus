---
title: "Event Sourcing: Integration Events"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Integration Events

*Internal events are your privates. Integration events are the denormalized, versioned contract you publish for other services.*

---

## Internal vs integration

An internal event names ids:

```
ItemMoved { productId: "24728347", from: "17", to: "28" }
```

The integration event for the same fact is denormalized — the consumer
gets the human-readable state it needs, not a join exercise:

```
ItemMoved {
  productId: "24728347", productName: "Peanut Butter", productSize: "16 oz",
  productLabel: "Jif Reduced Salt Peanut Butter", productBrand: "Jif",
  locationFromId: 17, locationFrom: "North Carolina Central Warehouse",
  locationToId: 28,   locationTo: "New York City Warehouse 2"
}
```

The typical shape: an **adapter** listens to internal events, denormalizes
data onto them, and publishes integration events to the outside. Several
adapters publishing different versions of the same integration event is
normal.

---

## Two rules

1. **Do not show others your privates.** Internal events are volatile —
   they exist to support your use cases and change as those change.
   Publishing them directly makes every consumer break on every internal
   refactor.
2. **Think forward.** An integration event may have 50-100 systems
   subscribed. Schema changes are not made; a **new** integration event is
   published alongside, and the old one deprecated over time (deprecation
   spans weeks to 5+ years, per organisation). Content-type negotiation
   for event bodies quickly earns its keep.

---

## Event aggregator

Not every consumer should understand every event. An **event aggregator**
watches the fine-grained internal events and publishes one coarse
"OrderUpdated"-style document carrying the current state:

```
OrderPlaced → FoodCooked → OrderPriced → OrderPaid
                        ↓ aggregator
        OrderUpdated { table: 2, total: 22.00, tip: 5.00, ... }
```

The consumer integrates with one contract instead of a stream of schemas.

---

## Negotiating event aggregator

Instead of embedding the state in the aggregated event, publish a **URI**
and let consumers negotiate the form:

```
FoodCooked { id: "24728347", uri: "https://mydomain.com/aggregator/24728347" }
```

The consumer fetches with content negotiation — format (JSON/XML) and,
critically, **version** — and the server converts to the version the client
prefers. Old versions stay supported, so hundreds of subscribers on
different update cycles never need to upgrade in lockstep. The explicit
questions: how long do we support a version, and what are the consumers'
release cycles?

---

## Direct event integration — and its ceiling

The simplest integration: service A listens to B's events and acts on
them. Perfectly fine while the service count is small and event volume
low. It fails as a complexity problem, not a performance one: at thirty
services, "who listens to what" becomes unmanageable, event chains become
invisible, and cross-team change coordination dominates.

It stays right in one classic case: **a single dataset nearly everything
needs** — market data in any financial institution. There, everyone
integrates with the one feed, levels and SLAs and all, because the data
is material to almost everything.

## Choosing

- Few services, low volume → direct integration, simplest thing that works.
- External consumers with their own update cycles → aggregator, then
  negotiating aggregator when versioning becomes the pain.
- The moment "who listens to what" needs a diagram that nobody can draw
  from memory, the direct approach has already stopped scaling.
