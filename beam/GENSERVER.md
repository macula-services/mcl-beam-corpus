---
title: "BEAM: GenServer"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: GenServer

*A process that owns a piece of state and changes it one message at a time. The workhorse behaviour of OTP.*

---

## What it is

Every BEAM process can keep state by passing it to itself: receive a
message, compute a new value, recurse with that value. `gen_server`
(Erlang) and `GenServer` (Elixir) package that loop once, correctly, with
the parts everyone would otherwise rewrite: synchronous request/reply
with timeouts, system messages, debugging hooks, clean shutdown and a
start function a supervisor can call. You supply only the callbacks
that say how the state reacts to each kind of message.

---

## The callbacks

```elixir
defmodule Inventory do
  use GenServer

  # Client API: plain functions, no pids or message shapes leak out
  def start_link(opts), do: GenServer.start_link(__MODULE__, %{}, opts)
  def stock(server, sku), do: GenServer.call(server, {:stock, sku})
  def receive_goods(server, sku, qty), do: GenServer.cast(server, {:receive, sku, qty})

  # Server callbacks
  @impl true
  def init(stock), do: {:ok, stock}

  @impl true
  def handle_call({:stock, sku}, _from, stock), do: {:reply, Map.get(stock, sku, 0), stock}

  @impl true
  def handle_cast({:receive, sku, qty}, stock),
    do: {:noreply, Map.update(stock, sku, qty, &(&1 + qty))}
end
```

| Callback | Runs when | Typical return |
|----------|-----------|----------------|
| `init/1` | The process starts; the caller of `start_link` waits for it | `{:ok, state}` (optionally with a timeout or `{:continue, arg}`) |
| `handle_call/3` | A client uses `GenServer.call/3` and blocks (default timeout 5000 ms) | `{:reply, reply, state}` |
| `handle_cast/2` | A client uses `GenServer.cast/2`; no reply, no delivery confirmation | `{:noreply, state}` |
| `handle_info/2` | Any other message: monitors, timers, raw `send`, the `:timeout` from a timeout return | `{:noreply, state}` |
| `handle_continue/2` | A callback returned `{:continue, arg}`; runs before the next message | `{:noreply, state}` |

---

## Using it well

- **Keep the API module thin and the logic pure.** The public functions
  hide `call`/`cast`; the callbacks delegate to plain functions over the
  state. The pure functions get unit tests without a process.
- **Pick call or cast deliberately.** `call` gives back-pressure and an
  answer; `cast` gives neither, so a fast producer can flood the mailbox.
  Default to `call` unless you truly do not care about the outcome.
- **Replying later is allowed.** A `handle_call` can return
  `{:noreply, state}`, keep the `from`, and answer with
  `GenServer.reply/2` once slow work (often a `Task`) finishes. The
  server stays free for other callers meanwhile.
- **Do slow setup in `handle_continue`,** not in `init/1`, so the
  supervisor that started the process is not blocked.

---

## Trade-offs and pitfalls

- **One process is one queue.** All access to the state is serialised,
  which is exactly what makes it consistent, and exactly what makes it a
  bottleneck under heavy read load. Read-mostly data often belongs in
  [ETS](ETS.md).
- **Do not use a process to organise code.** The Elixir docs put it
  plainly: use processes to model runtime properties (state,
  concurrency, failure), never for code organisation. Stateless logic is
  a module.
- **Keep a catch-all `handle_info/2`.** The default injected by
  `use GenServer` logs and drops unexpected messages; once you define
  your own `handle_info/2` that default is gone, and a message no clause
  matches crashes the server.
- **Simple value holders and one-off jobs** have lighter tools: see
  [TASKS_AND_AGENTS](TASKS_AND_AGENTS.md). Naming servers dynamically:
  [REGISTRY](REGISTRY.md). Restarting them: [SUPERVISION_TREES](SUPERVISION_TREES.md).

## Sources

- *Designing Elixir Systems with OTP*, 1st edition, James Edward Gray II and Bruce A. Tate, Pragmatic Bookshelf, 2019. <https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/>
- Elixir `GenServer` documentation. <https://elixir.hexdocs.pm/GenServer.html>
- Erlang/OTP `gen_server` reference. <https://www.erlang.org/doc/apps/stdlib/gen_server.html>
- Erlang/OTP design principles, gen_server behaviour. <https://www.erlang.org/doc/system/gen_server_concepts.html>
