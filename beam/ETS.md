---
title: "BEAM: ETS"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: ETS

*Erlang Term Storage: an in-memory key-value store built into the runtime. Fast lookups and counters, owned by a process, invisible to the GC's cost model.*

---

## What it is

ETS is a set of in-memory tables storing Erlang terms, created by a
process and accessed by any process (depending on protection). It is
the BEAM's answer to "I need a cache, a counter, a lookup table" —
without a database and without copying through message passing.

| Property | Value |
|----------|-------|
| Speed | Constant-time lookups/hashes, very fast reads |
| Storage | In memory; lost on owner exit unless an heir takes over |
| Ownership | A process owns the table (or `:public`/`:protected` access) |
| Scale | Not distributed — per node |

---

## Ownership and lifetime

A table is owned by the process that created it. When the owner dies,
the table dies with it — unless an **heir** was named:

```elixir
:ets.new(:my_table, [:named_table, {:heir, heir_pid, :inherited_data}])
```

Use the heir for caches that should survive the owner's restart. The
more common pattern is the opposite: let the table die with the process,
and rebuild it on start — ETS data is derived data.

---

## The table types

| Type | Behaviour |
|------|-----------|
| `:set` | One row per key (default) |
| `:ordered_set` | Sorted by key, ordered traversal |
| `:bag` | Many rows per key |
| `:duplicate_bag` | Many rows per key, duplicates allowed |

And the access modes: `:public` (any process), `:protected` (any process
may read, only the owner writes — the default), `:private` (owner only).

---

## What ETS is good at

- **Counters** — `:ets.update_counter/4` is atomic; concurrent increments
  do not race. The idiomatic hit counter.
- **Caches and registries** — lookup tables for config, sessions, routing.
- **Cross-process shared state** — faster than `Agent`/`GenServer`
  round-trips for read-heavy data.

## What ETS is not

- **Not durable.** Node restart, table gone. Persist elsewhere.
- **Not a database.** No transactions, no joins, no query language —
  `:ets.match` patterns only.
- **Not GC-free by magic.** Terms in ETS are copied on write; large
  binaries are ref-counted. Know the cost model before storing millions
  of rows.
- **Not distributed.** A table lives on one node; replicating it is your
  problem (Mnesia builds on ETS for this).

## Rules of thumb

- Prefer ETS over a `GenServer` holding a map when reads dominate: the
  `GenServer` serialises every read through one process; ETS lets all
  readers proceed in parallel.
- Prefer a `GenServer` when writes must be serialised with side effects;
  ETS alone gives no such ordering.
- Name the table, set the right access mode, and decide the heir policy
  *when you create it* — these are the three decisions that are painful
  to reverse later.
