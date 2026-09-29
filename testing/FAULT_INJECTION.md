---
title: "Testing: Fault Injection"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Fault Injection

*Kill processes in the running application and assert the system recovers. The tests that earn the BEAM's fault-tolerance claims.*

---

## The idea

Most test suites verify happy paths and error paths the developer
*expected*. Fault injection verifies the third category: what happens
when a process dies unexpectedly, mid-work, with no warning. On the
BEAM this is not exotic — it is the runtime's normal mode, and the
supervision tree is the code under test.

---

## Killing a process and asserting cleanup

```elixir
test "no file is left behind if the GenServer crashes" do
  path = Path.join(System.tmp_dir!(), Integer.to_string(System.unique_integer([:positive])))
  pid = start_supervised!({GenServerThatUsesFile, path: path})

  assert File.exists?(path)
  Process.exit(pid, :kill)                    # brutal, no cleanup callbacks run

  wait_for_passing(2_000, fn -> refute File.exists?(path) end)
end
```

The shape: start the process under test **supervised**
(`start_supervised!`), assert the pre-condition, kill it brutally, and
assert the post-condition — the cleanup ran, the state is consistent,
the world is whole again.

---

## wait_for_passing — the assertion that races

After `Process.exit(pid, :kill)` the cleanup happens *asynchronously* —
asserting immediately is a race condition. The standard helper retries
an assertion until it passes or the timeout runs out:

```elixir
defp wait_for_passing(timeout, fun) when timeout > 0 do
  fun.()
rescue
  _ -> Process.sleep(100); wait_for_passing(timeout - 100, fun)
end
defp wait_for_passing(_timeout, fun), do: fun.()
```

It returns as soon as the assertion passes, so passing tests stay fast;
the final iteration does not rescue, so a real failure fails the test.

---

## What to fault-inject

| Target | Assert |
|--------|--------|
| A worker | Its cleanup ran (files, locks, reservations released) |
| A supervised child | The supervisor restarted it, tree is whole again |
| The app mid-operation | No partial state: either the write completed or it did not |
| A projection mid-replay | Restart resumes at the checkpoint and converges (see [TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md)) |
| A crash loop | `max_restarts` trips, supervisor gives up, failure propagates as designed |

---

## The discipline

- **Kill with `:kill`, not a polite exit.** Polite exits run cleanup
  code; the point is to test the system with no such mercy.
- **Assert the outcome, not the logs.** "The supervisor restarted it"
  is the test for the tree; "the file is gone" is the test for the
  worker. Both matter; write both.
- **Keep the race explicit.** Every post-crash assertion is
  asynchronous; `wait_for_passing` makes the race a named part of the
  test instead of a flake.
- **Fault-inject the seams you depend on.** Test the checkpoints you
  rely on, the supervisors you designed, the cleanups you wrote — not
  the runtime's built-in guarantees.

## Why it matters

Supervision and let-it-crash are promises until a test proves them.
One fault-injection test per critical recovery path converts "we
designed for failure" into "we have shown the system recovers" — and
it is the cheapest insurance the BEAM offers, because the failure
machinery is already there to be exercised.
