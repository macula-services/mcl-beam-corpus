---
title: "BEAM: ETS"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: ETS

*Erlang Term Storage: in-memory tables built into the runtime. Fast shared lookups and atomic counters, owned by a process, stored outside process heaps.*

---

## What it is

ETS tables store tuples keyed on one element. A process creates a table
and, depending on the access mode, other processes can read or write it
directly, without a message round-trip to the owner. It is the BEAM's
answer to "I need a cache, a counter, a lookup table" without a
database and without funnelling every read through one process.

| Property | Value |
|----------|-------|
| Speed | `set` tables: constant-time lookup; `ordered_set`: logarithmic |
| Storage | In memory, outside process heaps, so not part of any process's garbage collection |
| Copying | Every insert and every lookup copies the term (large binaries are reference-counted instead) |
| Atomicity | Each update of a single object is atomic and isolated; there are no multi-operation transactions |
| Scope | One node; not replicated |

---

## Ownership and lifetime

The creating process owns the table. When the owner terminates the
table is destroyed, unless an **heir** was named at creation:

```elixir
:ets.new(:sessions, [:set, :public, :named_table, {:heir, keeper_pid, :sessions}])
```

The heir receives the table and an `{:"ETS-TRANSFER", tid, from, data}`
message. The more common design is the opposite: treat ETS contents as
derived data that is rebuilt when the owner restarts.

---

## Table types and access

| Type | Behaviour |
|------|-----------|
| `:set` | one object per key (default) |
| `:ordered_set` | one object per key, traversed in key order |
| `:bag` | many objects per key, no identical duplicates |
| `:duplicate_bag` | many objects per key, duplicates allowed |

| Access | Who may read | Who may write |
|--------|--------------|---------------|
| `:protected` (default) | any process | owner only |
| `:public` | any process | any process |
| `:private` | owner only | owner only |

`read_concurrency` and `write_concurrency` options tune locking for
read-heavy or write-heavy tables.

---

## Good uses

- **Counters:** `:ets.update_counter/3,4` increments atomically, so
  concurrent writers do not race.
- **Caches and lookup tables:** configuration, routing, session data.
- **Read-mostly shared state:** readers proceed in parallel instead of
  queuing on a [GenServer](GENSERVER.md). `Registry` is built on ETS
  ([REGISTRY](REGISTRY.md)).

## Limits

- **Not durable.** A node restart loses everything. (DETS and Mnesia
  add disk storage.)
- **Not a query engine.** Lookups by key, plus `match`/`select` with match
  specifications; no joins, no multi-key transactions.
- **Copy cost.** Storing or fetching large terms copies them every time.
- **Not distributed.** Replication is your problem (Mnesia builds on ETS
  for that).

## Rules of thumb

- Prefer ETS over a GenServer-held map when reads dominate.
- Prefer a GenServer when writes must be ordered with side effects;
  single-object atomicity is all ETS gives.
- Decide name, access mode and heir policy at creation; they are awkward
  to change later.

## Sources

- Erlang/OTP `ets` reference. <https://www.erlang.org/doc/apps/stdlib/ets.html>
