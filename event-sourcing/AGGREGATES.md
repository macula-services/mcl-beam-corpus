---
title: "Event Sourcing: Aggregates"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Aggregates

*An aggregate is a consistency boundary: the single place that decides whether a command is allowed, based only on its own history, and records the decision as events.*

---

## What it is

In domain-driven design an aggregate is a cluster of objects that is
changed as a unit, with one **root** that outsiders talk to. In an
event-sourced system it becomes the decision point of the write side:

```
command ──▶ aggregate ──▶ :ok + new events   (appended to its stream)
                │     └─▶ {:error, reason}   (nothing appended)
                └── current state = fold of its own stream
```

The aggregate never stores its current state as the truth. It rebuilds
it by folding its stream (optionally starting from a snapshot), which is
why it can be recreated on any node at any time.

## How it works

Splitting the aggregate into two pure functions keeps it easy to test:

```elixir
defmodule Showing do
  import Bitwise

  # status is an integer of bit flags, set with :evoq_bit_flags.set/2
  @closed 0b0001

  # decide: state + command -> events or refusal
  def execute(%{status: st}, %ReserveSeat{}) when band(st, @closed) != 0,
    do: {:error, :showing_closed}
  def execute(%{taken: taken}, %ReserveSeat{seat_id: s}) when is_map_key(taken, s),
    do: {:error, :seat_taken}
  def execute(_state, %ReserveSeat{seat_id: s, customer_id: c}),
    do: {:ok, [%SeatReserved{seat_id: s, customer_id: c}]}

  # evolve: state + event -> state (no validation, events are facts)
  def apply_event(state, %SeatReserved{seat_id: s, customer_id: c}),
    do: put_in(state.taken[s], c)
end
```

`execute` may refuse; `apply_event` must never refuse, because the event
already happened. The fold that rehydrates the aggregate uses only
`apply_event`.

## The root and the boundary

- Outside code refers to the aggregate by its id and sends commands to
  its root. It never reaches in and changes an inner entity directly.
- Inner entities (a seat inside a showing) need identity only within the
  aggregate.
- All rules that must hold *immediately and together* are checked inside
  the boundary, in one place, before any event is written.

## Rules of thumb

| Rule | Reason |
|------|--------|
| One stream per aggregate instance | The stream is the boundary; replaying it restores the whole |
| Decide on the complete, current history | A decision on a partial or stale fold is a wrong decision |
| Refusal is normal | Zero events is a valid outcome |
| Append with the expected version | If someone else appended first, the append fails; reload and retry, never overwrite |
| Keep aggregates small | Every command for one aggregate is serialised on one stream; big aggregates become hot spots |
| Refer to other aggregates by id | Rules spanning aggregates are eventually consistent, coordinated by a process manager ([SAGAS_AND_PROCESS_MANAGERS](SAGAS_AND_PROCESS_MANAGERS.md)) |

## Finding the boundary

- Start from the **invariants**: which facts must be true together, at
  the moment a command is accepted? Those belong in one aggregate.
  Everything else can be separate and eventually consistent.
- Name aggregates by the behaviour they protect. A thing that only holds
  fields and enforces nothing is a read model or a value, not an
  aggregate.
- Every command should have exactly one obvious owner. If none fits, the
  boundary is wrong or the command is really a multi-step process.

## On the BEAM

A common runtime shape is one process per live aggregate instance: a
GenServer registered by aggregate id, started on demand under a
DynamicSupervisor, folding its stream in `init/1` (or in
`handle_continue/2` so the caller is not blocked), and stopped after a
period of inactivity. The process mailbox serialises commands for that
instance; optimistic concurrency on append protects against a second
instance on another node. See [GENSERVER](../beam/GENSERVER.md) and
[SUPERVISION_TREES](../beam/SUPERVISION_TREES.md).

## Why it matters

Aggregates are where correctness lives. Projections, queries and
integrations only read what aggregates decided. Well-drawn boundaries
make the rest of the system derivation; badly drawn ones spread
contention and ambiguity to every consumer.

## Sources

- Alex Lawrence, *Implementing DDD, CQRS and Event Sourcing*, Leanpub, 2021 (now retired from sale on Leanpub). https://leanpub.com/implementing-ddd-cqrs-and-event-sourcing ; author's page: https://www.alex-lawrence.com/books/
- Vaughn Vernon, "Effective Aggregate Design" (three-part essay), 2011 (free PDFs). https://www.dddcommunity.org/library/vernon_2011/
- Martin Fowler, "DDD Aggregate", martinfowler.com, 2013 (free). https://martinfowler.com/bliki/DDD_Aggregate.html
- Eric Evans, *Domain-Driven Design Reference*, Domain Language, 2015 (free PDF). https://www.domainlanguage.com/ddd/reference/
