---
title: "BEAM: GenServer"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: GenServer

*The Generic Server: a process that loops, holding state, answering calls and casts. The fundamental abstraction for stateful processes in OTP.*

---

## What a GenServer is

A GenServer is a process running a loop that holds state:

```elixir
def loop(state) do
  receive do
    {:tick, _pid}      -> loop(Core.inc(state))
    {:state, pid}      -> send(pid, {:count, state}); loop(state)
  end
end
```

Every iteration receives a message, computes the **next** state, and
recurses with it. State lives in the recursion, not in a variable —
that is the whole trick. A GenServer is this loop, packaged with
callbacks, timeouts, and supervision integration, so you write the
state transitions instead of the loop.

---

## The callbacks

```elixir
defmodule Counter do
  use GenServer

  def init(state), do: {:ok, state}

  def handle_call(:get, _from, state), do: {:reply, state, state}
  def handle_cast({:tick, n}, state), do: {:noreply, state + n}
  def handle_info(msg, state), do: {:noreply, state}
end
```

| Callback | Triggered by | Return shape |
|----------|--------------|--------------|
| `init/1` | Start | `{:ok, state}` |
| `handle_call/3` | `GenServer.call/3` | `{:reply, reply, state}` — caller blocks for the reply |
| `handle_cast/2` | `GenServer.cast/2` | `{:noreply, state}` — fire and forget |
| `handle_info/2` | Any other message | `{:noreply, state}` — timeouts, monitors, plain sends |

Call = synchronous request/reply. Cast = asynchronous fire-and-forget.
`handle_info` is the back door: the process receives *something*, and
it is the `handle_info` clause that decides what it means — including
the `:timeout` message from `init`'s timer.

---

## The boundary shape

The idiomatic arrangement keeps the machinery behind a plain function
API:

```
Client ──▶ Counter.tick(pid)          # API: plain functions
              └─▶ GenServer.cast       # machinery, hidden
```

- **The module's public functions are the API**; they hide `call`/
  `cast` and pids from callers.
- **The callbacks are the server**; they hold the state transitions.
- This split is what lets you swap the process out for a pure module
  later, or wrap it in a supervisor without touching callers.

---

## Choosing: call, cast, or not-a-GenServer

| Need | Use |
|------|-----|
| Caller must know the result now | `call` |
| Fire and forget, no reply needed | `cast` |
| Reply that arrives later | `call` returns quickly; send a message back when done |
| No state at all | A plain module — **do not use a GenServer for stateless code** |
| One-off async work | `Task` — see [TASKS_AND_AGENTS](TASKS_AND_AGENTS.md) |

---

## Rules of thumb

- **Serialise through the process.** Everything that touches the state
  goes through messages; the mailbox is the queue that makes the state
  consistent.
- **Keep state transitions pure.** `handle_call` should compute the new
  state with a pure function; test that function directly, without the
  process.
- **Blocking work kills responsiveness.** A `call` that takes seconds
  blocks the server for every other caller. Heavy work belongs in a
  spawned `Task`, with the result messaged back.
- **The name `GenServer` fools nobody.** It is a process providing a
  service; "server" is baggage. Treat it as a state machine, not a
  service class.
