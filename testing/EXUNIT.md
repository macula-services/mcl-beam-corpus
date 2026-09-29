---
title: "Testing: ExUnit"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: ExUnit

*Elixir's built-in test framework. Structure each test in clear phases, group with `describe`, share setup through the context, and opt modules into concurrency with `async: true` when they touch no shared state.*

---

## Anatomy of a test

A useful way to read any test is Gerard Meszaros's four phases: set up,
exercise, verify, tear down.

| Phase | In ExUnit | Often absent when |
|-------|-----------|-------------------|
| Set up | `setup` callbacks, or the first lines of the test | The code under test is a pure function |
| Exercise | Calling the function or sending the message | Never |
| Verify | `assert`, `refute`, `assert_receive`, `assert_raise` | Never |
| Tear down | `on_exit/2`, or processes stopped automatically by `start_supervised` | Nothing shared was changed |

Two practical signals:

- If a test exercises and verifies, then exercises and verifies again,
  it is probably two tests.
- Tests that fail only sometimes usually leak state: something changed
  shared state (an application env value, a named process, a table) and
  never put it back.

## Grouping with describe

```elixir
defmodule Box.SeatMapTest do
  use ExUnit.Case, async: true

  describe "reserve/2" do
    test "marks a free seat as taken" do
      map = Box.SeatMap.new(["A1", "A2"])
      assert {:ok, map} = Box.SeatMap.reserve(map, "A1")
      assert Box.SeatMap.taken?(map, "A1")
    end

    test "refuses a seat that is already taken" do
      {:ok, map} = Box.SeatMap.new(["A1"]) |> Box.SeatMap.reserve("A1")
      assert {:error, :taken} = Box.SeatMap.reserve(map, "A1")
    end
  end
end
```

- `describe` blocks cannot be nested; one level of grouping keeps test
  names and setup easy to follow.
- A common convention is one `describe` per public function, with test
  names stating behaviour rather than implementation.
- Test names plus bodies are the most up-to-date documentation of how
  the code is meant to be called.

## Setup and context

```elixir
setup do
  map = Box.SeatMap.new(["A1", "A2", "A3"])
  %{map: map}
end

test "three free seats", %{map: map} do
  assert Box.SeatMap.free_count(map) == 3
end
```

- A `setup` callback returns a map or keyword list that is merged into
  the test **context**, which each test can pattern-match.
- `setup :name` or `setup [:a, :b]` runs named functions in order, each
  receiving and extending the context.
- `setup` inside a `describe` applies only to that group; `setup_all`
  runs once per module, in a separate process.
- For processes, prefer `start_supervised!/2`: the process is started
  under the test's supervisor and is guaranteed to be shut down before
  the next test starts. Use `on_exit/2` for other clean-up.

## Concurrency

```elixir
use ExUnit.Case, async: true
```

- `:async` **defaults to `false`**. Modules that set `async: true` run
  concurrently with other async modules; the tests *inside* one module
  still run one after another (Elixir 1.18's `:parameterize` option is
  the exception: parameter sets of an async module run concurrently).
- An async module must not depend on or modify global state: named
  processes, application env, the file system at fixed paths, a shared
  database without a sandbox. Leave such modules synchronous.
- Since Elixir 1.18, `use ExUnit.Case, async: true, group: :some_key`
  lets async modules that share one resource avoid running at the same
  time as each other while still running alongside everything else.

## Rules of thumb

- Turn on `async: true` wherever it is safe; a mostly-async suite on the
  BEAM is fast.
- Setup builds context; teardown restores the world. No shared state
  touched, no teardown needed.
- If an assertion is awkward to write, look at the API before bending
  the test.

See also [FAULT_INJECTION](FAULT_INJECTION.md) for tests that kill
processes, and [PROPERTY_BASED_TESTING](PROPERTY_BASED_TESTING.md).

## Sources

- Andrea Leopardi and Jeffrey Matthias, *Testing Elixir: Effective and Robust Testing for Elixir and its Ecosystem*, 1st edition, The Pragmatic Programmers, 2021. https://pragprog.com/titles/lmelixir/testing-elixir/
- ExUnit documentation, `ExUnit.Case` (async, describe, parameterize, group), hexdocs (free). https://hexdocs.pm/ex_unit/ExUnit.Case.html
- ExUnit documentation, `ExUnit.Callbacks` (setup, setup_all, on_exit, start_supervised!), hexdocs (free). https://hexdocs.pm/ex_unit/ExUnit.Callbacks.html
- Gerard Meszaros, "Four Phase Test", xUnit Patterns (free pattern page from *xUnit Test Patterns*, Addison-Wesley, 2007). http://xunitpatterns.com/Four%20Phase%20Test.html
