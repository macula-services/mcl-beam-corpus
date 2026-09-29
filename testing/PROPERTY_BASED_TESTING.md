---
title: "Testing: Property-Based Testing"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Property-Based Testing

*State the invariant, let generators produce the cases, and shrink failures to the smallest counterexample. PropEr (Erlang) and StreamData (Elixir) are the BEAM's tools.*

---

## Stateless vs stateful

Properties come in two flavours:

| | Stateless | Stateful |
|---|---|---|
| Shape | Generate inputs, check an invariant holds | Generate command *sequences*, check the system against a model |
| Fit | Pure, isolated components without side effects | Stateful integration: servers, registries, aggregates |
| BEAM example | A serializer's round-trip | A GenServer's behaviour under command sequences |

Stateless properties are the equivalent of unit tests; stateful
properties replace the integration tests that example-based testing
does poorly.

---

## The stateless shape

A property module has three sections: **properties**, **helpers**, and
**generators**.

```elixir
property "encode/decode round-trips" do
  forall term <- term_generator() do
    term == decode(encode(term))
  end
end
```

The framework expands the generators, runs the property against every
generated case, and reports. The property *is* the specification — the
invariant that must hold for all inputs the generators can produce.

---

## Shrinking — the diagnosis is the counterexample

A failing case is shrunk: the framework reduces the generated input
until it finds one that still fails but is as small as possible. The
book's cash-register example: a property fails with a register holding
a billion coins and a price in the hundreds of thousands — shrinking
finds the same failure with **$5 in the register and a cheap item**.

The minimal counterexample *is* the bug report: "the cash function
cannot make change when the customer's payment exceeds the register's
contents" — one line to debug instead of a data dump. When you design
a property, ask yourself what its minimal failing case would teach.

---

## Modeling — test against the obviously-correct

The strongest stateless trick: implement the same logic twice — the
real code and a **model** so simple it is obviously correct — and
assert they agree:

```elixir
property "biggest matches the model" do
  forall list <- list_generator() do
    Pbt.biggest(list) == model_biggest(list)
  end
end
```

The model is the oracle. The rule: the model must be *obviously
right* — if it shares the implementation's logic, you are comparing the
bug to itself. An even stronger form exists when a golden reference
implementation is available: compare against it directly.

---

## Stateful properties — state machine models

For stateful systems, model the component as a **state machine**: an
abstract model state, a set of commands with preconditions, and
postcondition checks. The framework generates valid command sequences
(shrinking sequences, not just data), drives the real system with
them, and checks the real state against the model state after each
command.

This is how property-based testing earns its keep on process-heavy
BEAM code — and the natural fit for event-sourced aggregates: the
model is the fold of events, the commands are the aggregate's
interface, and every sequence is a given/when/then test generated for
free. See [TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md).

---

## Tools

| Tool | Language | Notes |
|------|----------|-------|
| **PropEr** | Erlang | The original; stateful machinery most developed; usable from Elixir |
| **StreamData** | Elixir | Idiomatic generators, integrates with ExUnit |

Debugging generators is part of the workflow: `proper_gen:pick/1`
materialises an instance, `sample/1` shows a spread.

## When not to

- **Nondeterministic or side-effecting code.** Properties must be
  pure; mock the boundary first.
- **Where an example is the spec.** A regression for one observed bug
  is best kept as its exact example; write the property when a general
  invariant emerges.

## Rule of thumb

Stateless properties for codecs and pure folds; stateful models for
servers and aggregates. If the invariant is hard to state, the design
is hiding something — the property is a design tool, not just a test.
