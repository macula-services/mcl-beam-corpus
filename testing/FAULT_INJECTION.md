---
title: "Testing: Fault Injection"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Fault Injection

*Deliberately crash parts of a running system in a test and check that it returns to a correct state. On the BEAM this is how you turn "we designed for failure" into evidence.*

---

## Why

Ordinary tests cover the paths the author expected: success and handled
errors. Fault-injection tests cover the unexpected: a process dying in
the middle of its work, without warning and without running its own
clean-up. On the BEAM that is not an exotic scenario; supervisors,
links and monitors exist precisely for it, and they deserve tests like
any other code.

## The basic shape

1. Start the system under test the way production starts it (under a
   supervisor; in ExUnit, via `start_supervised!/2`).
2. Establish and assert the pre-condition.
3. Kill a process with `Process.exit(pid, :kill)`. The `:kill` reason
   cannot be trapped, so `terminate/2` does not run: nothing gets a
   chance to tidy up.
4. Wait for the death to be observed, then assert the post-condition.

```elixir
# async: false, because the process is registered under a global name
test "seat holder comes back with a fresh pid after a brutal kill" do
  start_supervised!({Box.SeatHolder, name: Box.SeatHolder})
  old = Process.whereis(Box.SeatHolder)
  ref = Process.monitor(old)

  Process.exit(old, :kill)
  assert_receive {:DOWN, ^ref, :process, ^old, :killed}

  eventually(fn ->
    new = Process.whereis(Box.SeatHolder)
    assert is_pid(new) and new != old
  end)
end
```

## Waiting without flakiness

Recovery is asynchronous, so asserting straight after the kill is a race.

- **Prefer a message.** Death is observable exactly: monitor the process
  and `assert_receive` the `:DOWN` message.
- **Poll only for effects that send no message** (a restart, a released
  lock, a deleted file), with a deadline:

```elixir
defp eventually(fun, timeout_ms \\ 1_000, interval_ms \\ 20) do
  deadline = System.monotonic_time(:millisecond) + timeout_ms
  do_eventually(fun, deadline, interval_ms)
end

defp do_eventually(fun, deadline, interval_ms) do
  fun.()
rescue
  error in [ExUnit.AssertionError] ->
    if System.monotonic_time(:millisecond) < deadline do
      Process.sleep(interval_ms)
      do_eventually(fun, deadline, interval_ms)
    else
      reraise error, __STACKTRACE__
    end
end
```

It returns as soon as the assertion passes, so green tests stay fast,
and after the deadline it re-raises the real assertion error so the
failure message is useful.

## Clean-up after a brutal kill

Because a killed process runs no code, clean-up that must survive
`:kill` has to live somewhere else: in a process that monitors the
worker, in the supervisor restarting it with fresh state, or in
resources the runtime releases automatically (ETS tables owned by the
process, Registry entries, links and monitors). A fault-injection test
is the only honest check that you put the clean-up in the right place.

## What to target

| Target | Assert |
|--------|--------|
| A worker holding a resource | The resource is released by whoever is responsible for it |
| A supervised child | It is restarted and the tree is complete again |
| A multi-step operation | No half-done state: either the whole write is visible or none of it |
| A projection mid-replay | After restart it resumes from its checkpoint and converges ([TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md)) |
| A child that keeps crashing | After the restart intensity is exceeded (Elixir's `Supervisor` default: 3 restarts in 5 seconds; Erlang's `supervisor` default: 1 in 5) the supervisor itself exits and the failure escalates as designed |

## Discipline

- **Use `:kill`.** With a trappable reason such as `:shutdown`, a
  process that traps exits gets to run its clean-up, so the test checks
  something gentler than a real crash. (`Process.exit(pid, :normal)`
  sent to another process does not stop it at all.)
- **Assert outcomes, not log lines.**
- **Make waiting explicit** with monitors or a deadline-based helper,
  never a bare `Process.sleep/1`.
- **Test your recovery design, not OTP.** You do not need to prove that
  supervisors restart children; you do need to prove that *your* child
  comes back in a correct state.

## Sources

- Andrea Leopardi and Jeffrey Matthias, *Testing Elixir: Effective and Robust Testing for Elixir and its Ecosystem*, 1st edition, The Pragmatic Programmers, 2021. https://pragprog.com/titles/lmelixir/testing-elixir/
- Elixir documentation, `Process.exit/2` and `Supervisor` (restart intensity defaults), hexdocs (free). https://hexdocs.pm/elixir/Process.html#exit/2 and https://hexdocs.pm/elixir/Supervisor.html
- Erlang/OTP documentation, `supervisor` (restart intensity and period), erlang.org (free). https://www.erlang.org/doc/apps/stdlib/supervisor.html
- ExUnit documentation, `ExUnit.Callbacks.start_supervised!/2`, hexdocs (free). https://hexdocs.pm/ex_unit/ExUnit.Callbacks.html
