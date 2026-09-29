---
title: "Testing: Property-Based Testing"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Property-Based Testing

*State the invariant, let generators find the counterexample, and shrink it to a minimal case. PropEr (Erlang) and StreamData (Elixir) are the BEAM's tools.*

---

## The idea

An example-based test checks one case. A property-based test checks a
**property** — an invariant that must hold for all inputs — against
hundreds of generated cases:

```elixir
property "encode/decode round-trips" do
  check all term <- term_generator() do
    term == decode(encode(term))
  end
end
```

When a generated case fails, the framework **shrinks** it: the
counterexample is reduced to a minimal failing case — the difference
between "some 200-element list broke it" and "the empty list breaks it".

---

## What it buys on the BEAM

- **Coverage of the space, not the sample.** The generator explores
  inputs a human test writer would never enumerate: malformed maps,
  mixed encodings, huge and tiny binaries.
- **The shrinking is the diagnosis.** A minimal counterexample is usually
  the bug report: the invariant holds everywhere except one edge.
- **Specifications instead of expectations.** Writing the property forces
  you to state what the code *should* do for everything, which is often
  harder than the implementation.

## The BEAM tools

| Tool | Language | Notes |
|------|----------|-------|
| **PropEr** | Erlang | The original; works from Elixir too |
| **StreamData** | Elixir | Idiomatic generators, integrates with ExUnit |

Both generate, run, and shrink automatically; both can test stateful
systems with state machine models (below).

---

## Where it pays most

1. **Serializers and codecs.** Round-trip properties over generated terms
   — the classic, and the highest hit rate.
2. **Pure domain logic.** An aggregate's `apply_event/2` fold: property
   "applying any valid event sequence leaves the aggregate in a valid
   state" catches illegal states no hand-written case would.
3. **Stateful models.** Model the system as a state machine, drive it
   with generated command sequences, assert the real system matches the
   model — this is how property-based testing earns its keep on
   process-heavy BEAM code.

## When not to

- **Nondeterministic or side-effecting code.** Properties must be pure;
  mock the boundary.
- **Where an example is genuinely the spec.** A regression for one
  observed bug is best kept as its exact example; the property version
  comes later if a general invariant emerges.

## Rule of thumb

Use property-based tests for codecs, pure folds, and state machines.
Pair them with the event-sourced test shapes: the aggregate fold is a
property-based test waiting to be written — [TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md).
