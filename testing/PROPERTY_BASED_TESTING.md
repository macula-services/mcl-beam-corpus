---
title: "Testing: Property-Based Testing"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Property-Based Testing

*Describe a rule that must hold for all inputs, let a generator produce hundreds of inputs, and when one breaks the rule, let the framework shrink it to a minimal counterexample. PropEr (Erlang, usable from Elixir via PropCheck) and StreamData (Elixir) are the BEAM's main tools.*

---

## The idea

An example-based test checks one input you thought of. A property states
something that must be true for *every* input of a certain kind, and the
framework searches for an input that makes it false. The approach comes
from Haskell's QuickCheck (Claessen and Hughes, 2000); Quviq QuickCheck
brought it to Erlang, and PropEr is the open-source Erlang tool in the
same family.

## A stateless property

The same round-trip property in the two main tools:

```elixir
# StreamData (use ExUnitProperties)
property "a seat code survives encode/decode" do
  check all row <- member_of(?A..?Z), number <- integer(1..60) do
    seat = %Seat{row: row, number: number}
    assert seat |> Seat.encode() |> Seat.decode() == {:ok, seat}
  end
end

# PropCheck, the Elixir wrapper for PropEr (use PropCheck)
property "a seat code survives encode/decode" do
  forall {row, number} <- {range(?A, ?Z), range(1, 60)} do
    seat = %Seat{row: row, number: number}
    Seat.decode(Seat.encode(seat)) == {:ok, seat}
  end
end
```

A property file usually has three parts: the properties, any helper
functions, and the generators that describe the input space. Getting the
generators right (realistic distribution, edge cases included) is often
most of the work. In PropEr, `proper_gen:pick/1` produces one value from
a generator and `proper_gen:sample/1` prints a spread, which helps when
debugging them.

## Useful kinds of property

| Kind | Shape | Example |
|------|-------|---------|
| Round trip | `decode(encode(x)) == x` | Serialisers, event upcasters |
| Invariant | Some fact holds after any operation | A seat map never has more taken seats than seats |
| Model / oracle | Real implementation agrees with a simple one | An optimised price calculation vs a naive, obviously correct one |
| Idempotence | `f(f(x)) == f(x)` | Normalising input, applying the same event twice to an idempotent projection |
| Metamorphic | A known change to input produces a known change to output | Adding a free seat increases the free count by one |

For the model kind, the model must be *obviously* right and must not
share code with the implementation, or you are only checking the bug
against itself. When replacing legacy code, the old implementation makes
an excellent model ([LEGACY_CODE](LEGACY_CODE.md)).

## Shrinking

When a property fails on a large random input, the framework repeatedly
tries simpler versions of that input (shorter lists, smaller numbers)
and keeps the simplest one that still fails. The report you get is not
"it broke on this 400-element list" but "it breaks on `[0, 0]`", which
usually points straight at the cause. When writing a generator, keep it
shrinkable (build from the library's combinators rather than drawing raw
random numbers) so this keeps working.

## Stateful properties

For systems with state (a GenServer, an aggregate, a cache) you describe
a **model** of the system as a state machine:

- an initial model state;
- commands that can be issued, each with a precondition saying when it
  is allowed;
- how each command changes the model state;
- a postcondition comparing the real system's response with what the
  model predicts.

The framework generates random valid *sequences* of commands, runs them
against the real system, checks every postcondition, and on failure
shrinks the sequence to the shortest one that still fails. In PropEr
this is `proper_statem` (callbacks `initial_state/0`, `command/1`,
`precondition/2`, `next_state/3`, `postcondition/3`), with `proper_fsm`
for finite-state-machine models; PropCheck exposes the same from Elixir.
StreamData does not provide stateful testing; use PropEr/PropCheck for
it.

This fits event-sourced aggregates well: the model is a simple fold of
the events the aggregate should emit, the commands are the aggregate's
commands, and every generated sequence is a given/when/then scenario
nobody had to write by hand
([TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md)).

## Tools

| Tool | Language | Notes |
|------|----------|-------|
| **PropEr** | Erlang | Stateless and stateful testing, integrates with Erlang type specs |
| **PropCheck** | Elixir | Wrapper around PropEr, including its stateful testing |
| **StreamData** | Elixir | Generators plus `ExUnitProperties` (`property`/`check all`), integrated with ExUnit; no stateful testing |

## When not to

- Code whose output depends on time, randomness or the outside world,
  until those are pushed behind a seam you control.
- A regression test for one specific reported bug: keep the exact
  example. Write a property once a general rule becomes clear.

## Rule of thumb

Stateless properties for codecs, pure functions and folds; stateful
models for processes and aggregates. If you cannot state the property,
you may not yet understand what the code is supposed to guarantee.

## Sources

- Fred Hebert, *Property-Based Testing with PropEr, Erlang, and Elixir: Find Bugs Before Your Users Do*, The Pragmatic Programmers, 2019. https://pragprog.com/titles/fhproper/property-based-testing-with-proper-erlang-and-elixir/ ; the author's free earlier draft: https://propertesting.com/
- PropEr documentation and API reference (`proper_statem`, `proper_gen`), free. https://proper-testing.github.io/ and https://proper-testing.github.io/apidocs/proper_statem.html
- StreamData documentation, `ExUnitProperties`, hexdocs (free). https://hexdocs.pm/stream_data/ExUnitProperties.html ; README (stateful testing not supported): https://github.com/whatyouhide/stream_data
- PropCheck documentation, hexdocs (free). https://hexdocs.pm/propcheck/readme.html
- Koen Claessen and John Hughes, "QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs", ICFP 2000 (free PDF copy in a Tufts course archive). https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf
