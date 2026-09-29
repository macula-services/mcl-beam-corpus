---
title: "BEAM: Concurrency Models"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Concurrency Models

*The BEAM's isolated processes with message passing are one answer to concurrency among several. Knowing the alternatives tells you when the BEAM fits and when to hand work to something else.*

---

## Two questions that separate the models

Every concurrency model answers two questions:

1. **Is state shared?** Either many threads touch the same memory and
   must coordinate, or each unit owns its data and others ask for it.
2. **What is the unit of composition?** A thread, a transaction, a
   channel, a process, or a whole data array.

| Family | State | Coordination | Where you meet it |
|--------|-------|--------------|-------------------|
| Threads with locks | shared, mutable | mutexes, condition variables, atomics | C, C++, Java, Rust |
| Immutable data + pure functions | shared but never mutated | none needed for reads; parallel maps and reducers | Haskell, Clojure, Elixir's own data |
| Software transactional memory | shared, mutated only in transactions | optimistic retry | Clojure refs, Haskell STM |
| Channels (CSP) | owned by goroutines/tasks | synchronous or buffered channels | Go, Clojure core.async, Kotlin |
| Actors / processes | owned by each process | asynchronous messages to a mailbox | Erlang, Elixir, Akka, Orleans |
| Data parallelism | large arrays | one kernel over all elements | GPUs via CUDA, OpenCL, Nx/EXLA |
| Batch + stream pipelines | immutable logs and derived views | framework-managed | MapReduce, Spark, Kafka Streams |

---

## Where the BEAM sits

The BEAM combines the process row with the immutable-data row: values
are immutable, and each process owns its heap, so the only way to
affect another process is a message. Three consequences follow:

- **Failure is local and observable.** A crash destroys one process's
  state and nothing else; links and monitors let another process react.
  That is the foundation of [SUPERVISION_TREES](SUPERVISION_TREES.md).
- **The same primitives span machines.** Send, link and monitor work
  across nodes ([DISTRIBUTION](DISTRIBUTION.md)), which shared-memory
  models cannot offer.
- **Fairness is built in.** Preemptive scheduling by reductions keeps
  latency stable under load ([SCHEDULER](SCHEDULER.md)).

## Actors versus channels

Both pass messages, and they are often confused. In CSP the **channel**
is the named thing: anonymous workers read and write channels, and a
program reads as a data-flow graph. In the actor model the **process**
is the named thing: messages go to an address (a pid or registered
name), each process has one mailbox, and a program reads as a set of
cooperating services. Channels make pipelines and back-pressure easy
to express; addressed processes make supervision and distribution easy.
On the BEAM, back-pressure comes from synchronous calls, demand-driven
libraries such as GenStage, or bounded `Task.async_stream`.

## When to reach for something else

- **Dense numeric work** (matrix math, training) wants data parallelism:
  call out to a GPU through a NIF or a library such as Nx, rather than
  spreading numbers across processes.
- **Tight shared-memory algorithms** with nanosecond budgets belong in
  native code; the BEAM's copying between processes costs more than a
  lock there.
- **Read-mostly shared lookups** inside a node are served by
  [ETS](ETS.md), a controlled exception to "nothing is shared".

## Sources

- *Seven Concurrency Models in Seven Weeks: When Threads Unravel*, 1st edition, Paul Butcher, Pragmatic Bookshelf, 2014. <https://pragprog.com/titles/pb7con/seven-concurrency-models-in-seven-weeks/>
- Erlang/OTP Reference Manual, Processes. <https://www.erlang.org/doc/system/ref_man_processes.html>
- Elixir `Task` documentation (`async_stream/3`). <https://elixir.hexdocs.pm/Task.html>
