---
title: "BEAM: Tasks and Agents"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Tasks and Agents

*Task: run something async, get the result. Agent: hold simple state. Both are GenServers with ergonomic fronts — reach for them before a raw GenServer.*

---

## Task — async computation

A Task runs a function in a process and delivers the result to the
caller:

```elixir
task = Task.async(fn -> do_heavy_work() end)
# ... other work ...
result = Task.await(task, 15_000)
```

| Function | Does |
|----------|------|
| `Task.async/1` + `Task.await/2` | Run concurrently, collect the result |
| `Task.start/1` | Fire and forget |
| `Task.async_stream/3` | Concurrent, backpressured, ordered results over a collection |
| `Task.Supervisor.async/2` | As `async`, but children are supervised |

The contract: the task is a **one-shot computation**, not a long-lived
state holder. It exits when the function returns; nothing survives it
by design.

---

## Agent — simple state

An Agent wraps a value the way a GenServer wraps a state machine, for
the case where all you need is "get and update this thing":

```elixir
{:ok, pid} = Agent.start_link(fn -> 0 end)
Agent.update(pid, &(&1 + 1))
Agent.get(pid, & &1)
```

The update function runs **inside the agent's process**, so updates
are serialised — but the function should stay fast and pure for the
same reasons a GenServer callback must.

---

## The hierarchy of choosing

| Need | Tool |
|------|------|
| One async computation | `Task` |
| A value that must be read/updated safely | `Agent` |
| State with a lifecycle, many message kinds, timeouts | `GenServer` |
| Dynamic children on demand | `DynamicSupervisor` + `Task.Supervisor` |

## Rules of thumb

- **Task for the work, not the state.** If the task result must live
  on, store it somewhere supervised — the task dies with its result.
- **Agent for data, GenServer for behaviour.** When the updates need
  rules beyond "apply this function", you have a behaviour, and a
  GenServer's callbacks are where the rules go.
- **`async_stream` is the production shape.** For processing
  collections concurrently, it gives you backpressure and supervision
  that a raw spawn loop does not.
- **Never `Task.async` without an await path.** An unawaited `async`
  task is a process leak; `start` (fire and forget) is the honest
  spelling of "I will not collect this result".
