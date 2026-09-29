---
title: "BEAM: The Scheduler"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: The Scheduler

*One OS process, a few scheduler threads, millions of BEAM processes. The scheduler is why the BEAM stays responsive under load.*

---

## The m:n model

The entire VM runs as a **single OS process** with a small number of OS
threads. By default the BEAM starts **as many schedulers as there are CPU
cores**: four cores, four schedulers. Each scheduler runs in its own OS
thread and takes turns running BEAM processes:

```
OS process
├── scheduler thread 1 ──▶ process A ──▶ process D ──▶ ...
├── scheduler thread 2 ──▶ process B ──▶ process E ──▶ ...
├── scheduler thread 3 ──▶ process C ──▶ ...
└── scheduler thread 4 ──▶ ...
```

Each process gets an execution time slot; when it is up, the running
process is **preempted** and the next one takes over.

---

## Why processes are cheap

- A process takes **a couple of microseconds** to create and starts at a
  few kilobytes of memory. An OS thread costs megabytes just for the
  stack.
- The VM's theoretical process limit is roughly **134 million**.
- The cost model that follows: use a dedicated process per task. Each
  long-running query, connection, or worker gets its own process, and
  all CPU cores stay busy without the developer managing threads.

---

## Reductions — the unit of work

The scheduler accounts work in **reductions**: roughly, function calls
and small units of execution. A process runs until it has used its
reduction budget (or blocks on a receive), then yields. Reductions are
what make preemption fair: a CPU-bound process cannot monopolise a
scheduler, because it is interrupted by budget, not by cooperation.

You can see a process's reductions with `Process.info/2` — the
`reductions` field is "the number of instructions this process has
executed".

---

## Concurrency vs parallelism

The scheduler makes the distinction precise:

- **Concurrency** is independent execution contexts. Five concurrent
  queries on one core take ten seconds total, exactly like sequential
  execution — no speedup.
- **Parallelism** is speedup from more cores. Same five queries, four
  schedulers: roughly a quarter of the time.

The BEAM gives you concurrency for free; parallelism arrives when cores
do. Structure for concurrency, scale by adding cores.

---

## Tuning notes

- Defaults are right almost always. Change them only with a reason:
  `+S N` pins the scheduler count; `System.schedulers/0` reports it.
- **Fewer schedulers** can help when CPU contention from other OS
  processes dominates — fewer BEAM threads, less thrash.
- **Dirty schedulers** (separate threads for blocking NIFs) exist for
  CPU-bound native work; long-running NIFs belong on the dirty side or
  they block a normal scheduler.

## Why it matters

The scheduler is the property that makes "let it crash" affordable: a
misbehaving process is preempted by budget, isolated by design, and
restarted by supervision — and the other million processes never feel
it. Everything else on the BEAM builds on that guarantee.
