---
title: "BEAM: The Scheduler"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: The Scheduler

*The runtime multiplexes many lightweight processes onto a few OS threads and preempts them by work done. That is why one busy process cannot freeze a node.*

---

## What it is

A running BEAM node is one OS process. Inside it, the runtime starts one
**scheduler thread per logical processor** by default (tunable with the
`+S` emulator flag; `System.schedulers_online/0` or
`:erlang.system_info(:schedulers_online)` report it). Each scheduler has
a run queue of BEAM processes and executes them one at a time. Idle
schedulers steal work from busy ones, so load spreads without the
programmer placing anything.

BEAM processes are not OS threads. They are runtime objects with their
own small heap and stack, created in microseconds. In current OTP the default limit is
1,048,576 simultaneous processes, raisable with `+P` up to 134,217,727.
Hundreds of thousands per node is ordinary.

---

## How preemption works

Work is counted in **reductions**, roughly one per function call plus
extra for built-in functions and I/O. A process runs until it has spent
its reduction budget (a few thousand), or blocks in `receive`, and then
goes back to the queue. Because the budget is counted by the runtime,
not surrendered by the code, a tight loop cannot starve its neighbours.
`Process.info(pid, :reductions)` shows how much work a process has done;
comparing two samples is a quick way to find the busy one.

The exception is native code. A NIF runs outside that accounting, so a
long NIF call blocks its scheduler. Long-running native work must go on
**dirty schedulers** (separate thread pools for CPU-bound and I/O-bound
NIFs) or be split into short calls.

---

## Concurrency is not parallelism

- **Concurrency:** many independent activities in progress. The BEAM
  gives you this on any machine, even one core, and it keeps latency
  fair: a slow request does not delay a fast one.
- **Parallelism:** activities running at the same instant. You get as
  much as you have schedulers with work. Ten CPU-bound jobs on four
  cores finish in roughly the time of three rounds, not one.

Design for concurrency (one process per independent activity); the
scheduler turns cores into parallelism when they exist.

---

## Tuning and pitfalls

- **Leave the defaults** unless measurement says otherwise. In
  containers with a CPU quota, check that the scheduler count matches
  the quota rather than the host's core count.
- **Low load average can hide throttling:** a node pinned at its cgroup
  CPU cap may look calm; read the cgroup CPU statistics.
- **Shared bottlenecks defeat the scheduler:** a single
  [GenServer](GENSERVER.md) everyone calls serialises work no matter
  how many cores exist. See [ETS](ETS.md) for read-heavy shared data.
- **Why it matters for fault tolerance:** isolation plus preemption means
  a misbehaving process can be killed and restarted by its supervisor
  ([SUPERVISION_TREES](SUPERVISION_TREES.md)) while everything else
  keeps its latency. Compare other runtimes in [CONCURRENCY_MODELS](CONCURRENCY_MODELS.md).

## Sources

- *Elixir in Action*, 3rd edition, Saša Jurić, Manning, 2024. <https://www.manning.com/books/elixir-in-action-third-edition>
- Erlang/OTP `erl` command reference (`+S`, `+P`, dirty scheduler flags). <https://www.erlang.org/doc/apps/erts/erl_cmd.html>
- Erlang/OTP `erlang` module reference (`process_info/2`, `system_info/1`). <https://www.erlang.org/doc/apps/erts/erlang.html>
- Elixir `System` documentation (`schedulers_online/0`). <https://elixir.hexdocs.pm/System.html>
