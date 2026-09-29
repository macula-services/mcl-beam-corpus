---
title: "BEAM: Concurrency Models"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Concurrency Models

*The actor model is one of seven ways to handle concurrency. Know the landscape to know when the BEAM's answer is — and is not — the right one.*

---

## The seven models

| Model | Core idea | Language/framework |
|-------|-----------|--------------------|
| **Threads and locks** | Formalise what the hardware does | Java, C++, everywhere |
| **Functional programming** | Immutability removes the shared-state problem | Clojure, Haskell |
| **CSP** | Message passing *through channels*, first-class | Go, Clojure core.async |
| **Actors** | Message passing *between processes*, each with state | Erlang/Elixir, Akka |
| **STM + agents** | Transactions over shared state, atomic updates | Clojure refs/agents |
| **Data parallelism** | One operation, many data elements — the GPU | OpenCL, CUDA |
| **Lambda architecture** | Batch + stream layers over the same data | Big Data stacks |

---

## The actor model's sweet spot

The BEAM's model is distinguished by three properties the others do
not combine:

1. **Fault tolerance.** Actors provide sophisticated error detection and
   recovery: processes are isolated, monitored, and restarted by
   supervision. Threads give you none of this.
2. **Distribution for free.** The actor model targets shared- *and*
   distributed-memory architectures; the same code runs on one node
   or twelve.
3. **Shared state, contained.** Each actor owns its state; nothing
   outside can touch it. Functional programming avoids shared state
   entirely; actors *contain* it.

---

## When each model wins

| Situation | Model |
|-----------|-------|
| Long-lived, resilient services; many independent stateful things | Actors (the BEAM) |
| Pipelines and stream processing, backpressure | CSP channels |
| Massive numeric workloads on one box | Data parallelism (GPU) |
| Highly concurrent reads over mostly-shared data | STM (or ETS-style structures) |
| Terabytes of batch data | Lambda architecture |
| Low-level, performance-critical control | Threads and locks — with care |

## The comparison that matters

**CSP vs actors** is the classic confusion — both are message passing.
The difference: CSP puts the **channel** first (communication is the
abstraction; processes are incidental), actors put the **process** first
(channels do not exist; processes are the abstraction). CSP programs
read as data flows; actor programs read as conversations between
independent beings. The BEAM's supervision tree has no CSP
equivalent — and CSP's composable pipelines have no direct BEAM
equivalent either.

## Rules of thumb

- Reach for another model only when the BEAM's is genuinely wrong
  for the work — the BEAM covers most server-side concurrency.
- GPU-bound number crunching belongs in NIFs/ports, not in actors.
- If you find yourself building channels and pipelines *inside* Elixir,
  you are writing CSP in an actor language; either accept the
  impedance mismatch deliberately or use the right tool.
