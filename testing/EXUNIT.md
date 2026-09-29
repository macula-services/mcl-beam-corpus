---
title: "Testing: ExUnit"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: ExUnit

*Elixir's built-in test framework: four phases per test, describe for grouping, async by default, and the suite as documentation.*

---

## The four phases

Every test decomposes into the same stages — **setup, exercise,
verify, teardown**:

| Phase | What happens | Notes |
|-------|--------------|-------|
| Setup | Prepare inputs or shared state | Often absent for purely functional code |
| Exercise | Call the code under test | Always present |
| Verify | Assert the behaviour | Always present |
| Teardown | Restore shared state | Absent when nothing was shared |

Two signals worth internalising:

- A **second recurrence of a phase** in a unit test means the test is
  doing too much — split it.
- "Random" failures are almost always a missing **teardown**: a test
  changed shared state and never restored it.

---

## Organising with describe

```elixir
describe "parse_response/1" do
  test "parses a valid response" do
    ...
  end
end
```

- `describe` groups tests that share a purpose (and, below, a setup).
- ExUnit allows **one level** of grouping — no nesting. The flatness
  is deliberate: force tests to stay readable.
- The suite's secondary purpose is **documentation**: text descriptions
  of expected behaviour plus working examples of how to call the code.
  Write the descriptions for the future you.

---

## Setup blocks

```elixir
setup do
  [user: create_user()]
end

test "the user has a name", %{user: user} do
  assert user.name
end
```

The setup block's return value becomes the test's **context** map.
Shared setup goes in `setup`; per-test setup stays in the test. `setup
[:fun, :other]` chains named setup functions in order.

---

## async: true — the default that matters

```elixir
use ExUnit.Case, async: true
```

Tests run **concurrently** per module when async is on — the BEAM
making test suites fast for free. The contract: async tests must not
touch shared state. If a test (or a whole module) does — a global
registry, a singleton process, a shared database — turn async off for
*that* module. Mixing is normal: async modules by default, `async:
false` on the ones that share.

## Rules of thumb

- **One describe per function under test**, tests named for behaviour,
  not implementation.
- **Setup returns context, teardown restores world.** If there is
  nothing to restore, you have no teardown — and a good test.
- **Async everywhere it is safe.** Serialise only what genuinely
  shares state; the suite speed is worth the discipline.
- **The assertion is the spec.** If an assertion is hard to write, the
  API is hard to use — fix the API before papering over the test.
