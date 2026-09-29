---
title: "Testing: Event-Sourced Systems"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Event-Sourced Systems

*When state is a fold over events, tests can be written as history: given these past events, when this happens, then expect these new events or this read model. Such tests double as a readable specification.*

---

## Why event sourcing changes testing

In a state-based system, a test sets up rows in a database and inspects
rows afterwards. In an event-sourced system the natural inputs and
outputs are events. That has two consequences:

- Tests can be phrased in the language of the business ("given the seat
  was reserved and then paid for, when the customer cancels, then a
  refund is issued"), so the suite reads as a specification of
  behaviour that domain experts can check.
- The write side needs no database, no projections and no mocks to be
  tested: the aggregate's decision is a pure function of its history and
  the command.

## Three kinds of test

### 1. Command tests: given, when, then

```elixir
test "a paid reservation that is cancelled is refunded" do
  given = [%SeatReserved{seat_id: "B-12", customer_id: "c1"},
           %PaymentCaptured{seat_id: "B-12", amount_cents: 2_400}]

  state = Enum.reduce(given, Showing.initial(), &Showing.apply_event(&2, &1))

  assert {:ok, [%RefundIssued{seat_id: "B-12", amount_cents: 2_400},
                %SeatReleased{seat_id: "B-12"}]} =
           Showing.execute(state, %CancelReservation{seat_id: "B-12"})
end
```

Also write the refusals: given a history, when a command arrives, then
`{:error, reason}` and no events. These are the workhorse tests of the
write side. A tiny helper that takes `given`, `when` and `then` keeps
them to a few lines each.

### 2. Projection tests: given events, expect a read model

Feed a list of events through a projection's handler and assert on the
resulting read model. Then run the same list again, or replay from an
earlier checkpoint, and assert the read model is unchanged: handlers
must be idempotent because delivery is usually at least once
([PROJECTIONS](../event-sourcing/PROJECTIONS.md),
[CHECKPOINTS](../event-sourcing/CHECKPOINTS.md)).

### 3. Recovery tests: crash, restart, converge

Kill a projection or process manager part way through a replay, let the
supervisor restart it, and assert that it resumes from its checkpoint
and reaches the same read model as an uninterrupted run. See
[FAULT_INJECTION](FAULT_INJECTION.md) for the mechanics.

## Why "given events" is better than fixtures

- You cannot set up a state that no sequence of events could produce, so
  tests never rely on impossible data.
- Read-model schema changes do not break write-side tests: those tests
  only know events.
- A test's history explains the situation. The order of events in
  `given` is part of the scenario, which a row fixture hides.

## Going further

- **Property-based tests** generate the histories for you: random valid
  command sequences run against the aggregate and a simple model
  ([PROPERTY_BASED_TESTING](PROPERTY_BASED_TESTING.md)).
- **Upcaster tests**: for every old event version still in the store,
  assert it decodes into the current shape. These guard replay after a
  schema change.
- **Fast feedback matters more than usual.** A mistake in an event's
  shape, once written to the store, is permanent and every consumer has
  to cope with it. Catch it in a test minutes after writing it, not after
  deployment.

## Rule of thumb

Command tests for every command (including refusals), projection tests
for every projection (including replay and duplicates), and at least one
recovery test per kind of checkpoint you rely on. If a scenario cannot
be written as "given events", the design is keeping state somewhere the
events do not explain.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Microsoft, "Event Sourcing pattern" (testing considerations), Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing
- Microsoft patterns & practices, *Exploring CQRS and Event Sourcing* (Journey 4 discusses the team's testing approach), 2012 (free online). https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10)
