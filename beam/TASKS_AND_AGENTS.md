---
title: "BEAM: Tasks and Agents"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Tasks and Agents

*Two small Elixir abstractions over processes: a Task runs one job, an Agent holds one value. Use them when a full GenServer would be ceremony.*

---

## Task: one job in its own process

```elixir
reports = Task.async(fn -> Reports.render(month) end)
totals  = Ledger.totals(month)          # runs meanwhile
Task.await(reports)                     # default timeout 5000 ms
```

| Function | Linked to caller | Result |
|----------|------------------|--------|
| `Task.async/1` + `Task.await/2` | yes (also monitored) | returned to the caller, who must await it |
| `Task.start/1` | no | none: fire and forget |
| `Task.start_link/1` | yes | none; usually a supervised child |
| `Task.async_stream/3` | yes | a stream of results, `max_concurrency` defaulting to the number of online schedulers, ordered by default |
| `Task.Supervisor.async_nolink/3` | no (monitored) | `Task.yield/2` gives `{:ok, result}` or `{:exit, reason}`; the caller survives a crashing task |

A task lives exactly as long as its function. Its result is a message to
the caller, and the reply is always sent, so an `async` that is never
awaited leaves a stray message in the caller's mailbox.

---

## Agent: one value behind a process

```elixir
{:ok, seen} = Agent.start_link(fn -> MapSet.new() end)
Agent.update(seen, &MapSet.put(&1, "node-3"))
Agent.get(seen, &MapSet.size/1)
```

The functions you pass run **inside the agent process**, so updates are
serialised and atomic. That cuts both ways: expensive work inside the
agent blocks every other client, while pulling the state out to compute
on it gives up atomicity. Choose per operation.

---

## Picking the tool

| Need | Tool |
|------|------|
| Run something concurrently and use its result | `Task.async` / `await` |
| Process a collection concurrently with bounded parallelism | `Task.async_stream` |
| Background job whose failure must not kill the caller | `Task.Supervisor` |
| A shared value with get/update semantics | `Agent` |
| State with rules, several message kinds, timers, monitors | [GENSERVER](GENSERVER.md) |
| Long-lived workers started on demand | `DynamicSupervisor` ([SUPERVISION_TREES](SUPERVISION_TREES.md)) |

## Pitfalls

- **Linked tasks crash the caller.** `Task.async` is linked; use
  `Task.Supervisor.async_nolink` when the job may fail and the caller
  must continue.
- **A task result is not stored anywhere.** If it must outlive the
  caller, hand it to a process or table that is supervised.
- **An Agent that grows rules is a GenServer in disguise.** Move to
  explicit callbacks once updates need validation or side effects.
- For CPU-heavy fan-out, remember the scheduler count bounds real
  parallelism ([SCHEDULER](SCHEDULER.md)).

## Sources

- *Designing Elixir Systems with OTP*, 1st edition, James Edward Gray II and Bruce A. Tate, Pragmatic Bookshelf, 2019. <https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/>
- Elixir `Task` documentation. <https://elixir.hexdocs.pm/Task.html>
- Elixir `Task.Supervisor` documentation. <https://elixir.hexdocs.pm/Task.Supervisor.html>
- Elixir `Agent` documentation. <https://elixir.hexdocs.pm/Agent.html>
