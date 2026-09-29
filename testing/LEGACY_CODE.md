---
title: "Testing: Legacy Code"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Legacy Code

*Changing code that has no tests: pin down what it does today, open a seam where you need to change it, and make the change under that safety net. Test around the change, not the whole system.*

---

## What "legacy" means here

Michael Feathers's working definition is the useful one: legacy code is
code without tests. Age is irrelevant. Without tests you cannot tell
whether a change broke something, so every change is a gamble, and the
testing problem and the change problem are one problem.

## Seams

A **seam** (Feathers's term) is a place where you can change what the
code does without editing the code at that place. In Elixir and Erlang
the common seams are:

| Seam | Example |
|------|---------|
| A function argument | Pass the clock, the HTTP client or the repo in, instead of calling it directly |
| Application config | `Application.get_env(:box, :payments, Box.Payments.Live)` chooses an implementation |
| A behaviour | Code calls `impl().charge/2`; tests supply a module implementing the same behaviour (Mox builds these) |
| A process boundary | Code sends messages to a registered name or a pid it was given; a test starts a stand-in process under that name |
| A module boundary | Extract the awkward call into your own module, which becomes the thing you substitute |

Untested code tends to have its seams welded shut: direct calls to
external systems, hard-coded names, logic buried inside a large
`handle_call/3`. The work is to open just enough of them.

## The loop

1. **Start from the change you actually need**, not a wish to clean up.
2. **Find the smallest area** around that change that you can test.
3. **Break the dependencies** that stop you calling it from a test,
   using the least invasive seam available (often: extract a function,
   add a parameter).
4. **Write characterisation tests** for the current behaviour of that
   area.
5. **Make the change**, adding tests for the new behaviour. The
   characterisation tests tell you what else you affected.
6. Optionally refactor, still under the tests.

## Characterisation tests

A characterisation test records what the code *does*, not what it
*should* do. Call it, observe the result, and assert exactly that,
surprises included:

```elixir
test "discount rounds half-cents down (current behaviour, see issue #212)" do
  assert Box.Pricing.discounted(1_005, 0.5) == 502
end
```

- Write them before refactoring. Otherwise you cannot tell a behaviour
  change from a refactor.
- When the observed behaviour looks like a bug, still pin it, name it in
  the test, and decide separately whether to fix it. Somebody may
  depend on it.
- Snapshot-style ("golden master") tests over a wide range of inputs are
  a quick way to characterise a pure function you do not yet understand.
  Property-based testing against the old implementation as a model is a
  stronger version ([PROPERTY_BASED_TESTING](PROPERTY_BASED_TESTING.md)).

## On the BEAM

Processes help. A GenServer's public API is already a boundary: if the
legacy module is a process, a test can start it with `start_supervised!`
and drive it through its API, or start a substitute under the same name
so that its callers can be tested. Logic tangled inside callbacks can be
moved into plain functions that the callbacks call, which are then
testable without any process at all.

## Rules of thumb

- Aim for a safe change, not a fully tested legacy system.
- Each seam you open lowers the cost of every later change nearby.
- Characterise first, then refactor, then add the feature.

## Sources

- Michael C. Feathers, *Working Effectively with Legacy Code*, Prentice Hall (Pearson), 2004. https://www.informit.com/store/working-effectively-with-legacy-code-9780131177055
- Nick Chamberlain, *Intuitive Testing with Legacy Code*, self-published via buildplease.com (sample dated 2016; ASP.NET MVC examples). https://buildplease.com/products/itmvc/
- Martin Fowler, "Legacy Seam", martinfowler.com, 2024 (free). https://martinfowler.com/bliki/LegacySeam.html
- "Characterization test", Wikipedia (free overview; term attributed to Feathers). https://en.wikipedia.org/wiki/Characterization_test
