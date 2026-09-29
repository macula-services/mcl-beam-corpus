---
title: "Testing: Event-Sourced Systems"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Event-Sourced Systems

*Event sourcing changes what tests are for: they are not just validation, they are the documentation of what the system does.*

---

## The shift in mindset

In an event-sourced system, tests stop being a detail and become the
**communication mechanism** for the system's behaviour. A test suite
written as "given these events, when this command, then these events" is
a machine-checked specification — and the test run is a use-case
documentation artifact.

This reverses the usual relationship: the tests *are* the doc, and the
doc is executable.

---

## The three test shapes

### 1. Command tests — "given, when, then"

```
given:  [ItemCreated, ItemPriced]        # prior events, hydrated
when:   MarkItemPaid                     # the command
then:   [ItemMarkedPaid]                 # expected new events
```

The aggregate's history goes in, the command goes in, the events come
out. No database, no projection, no mocks — the aggregate is a pure
function over its history. This is the workhorse test of the write side.

### 2. Replay tests — "given events, the read model is X"

```
given:  [OrderPlaced, OrderPaid, OrderShipped]
project: OrderStatusProjection
expect: %{order_id: 77, status: :shipped}
```

Feed an event sequence to a projection and assert the resulting read
model. Then feed the same sequence again from a restored checkpoint and
assert **idempotency** — the read model ends identical, nothing doubles.

### 3. Fault tests — "kill it, it recovers"

Kill the projection mid-replay, restart it, assert it resumes at the
checkpoint and converges to the same read model. These are the tests
that earn the fault-tolerance claims the architecture makes.

---

## Why "given events" beats fixtures

A traditional test sets up state by writing rows. An event-sourced test
sets up state by *listing the events that produced it*. The difference:

- The setup **is** part of the specification — you cannot create state
  the events cannot explain.
- Tests survive schema changes on the read side: they only know events,
  and the read models are rebuilt anyway.
- A regression reads as history: "given the order was paid *then*
  shipped, when the customer cancels..." — the why is in the given.

---

## Failing fast

The point of automated tests is to find mistakes **as fast as possible**:
fifteen seconds after a change, not three days later in manual testing.
The longer a bug lives, the more code becomes dependent on its
behaviour — some bugs cost weeks to remove not because they are hard but
because a hundred things rely on them. In an event-sourced system the
equivalent failure mode is a bad *event* that every projection learns to
work around; the power-cable test at event-design time is cheaper than
every consumer's workaround.

## Rule of thumb

Write the **given/when/then** command tests for every command, replay
tests for every projection, and at least one fault test per checkpoint
kind you rely on. If the test cannot be written as "given events", the
design is hiding state from itself.
