---
title: "BEAM: Registry"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Registry

*A local, decentralized way to name processes: register a key, look it up by key. No global name server, no single point of failure.*

---

## What it solves

Processes need to find each other. Options:

| Approach | Shape | Problem |
|----------|-------|---------|
| Pass pids around | `Counter.tick(pid)` | Pids die with the process; callers hold stale values |
| Global name registration | `Process.register/2`, `:global` | One name per atom, one registry per node (or cluster-wide churn) |
| **Registry** | `Registry.lookup(MyRegistry, key)` | Many registries, per-node, keyed by any term |

A Registry is a process that maps keys to pids — and it is not
fragile: lookups do not involve the registry process itself, and it
works with ETS under the hood.

---

## Using it

```elixir
Registry.start_link(keys: :unique, name: MyRegistry)   # part of the app tree

{:ok, _} = Registry.register(MyRegistry, "session-42", %{user_id: 7})
[{pid, meta}] = Registry.lookup(MyRegistry, "session-42")
Registry.unregister(MyRegistry, "session-42")
```

| `keys:` | Behaviour |
|---------|-----------|
| `:unique` | One process per key — the typical case |
| `:duplicate` | Many processes per key — e.g. all handlers of an event type |

The value stored with `register/3` (the metadata) rides along in
lookups — no second call needed.

---

## The via pattern

OTP name registration accepts `{:via, Registry, {reg, key}}`:

```elixir
GenServer.start_link(Mod, arg, name: {:via, Registry, {MyRegistry, "session-42"}})
```

Now the process is reachable by key through every OTP API that accepts
a name — supervisors, `GenServer.call`, `Process.whereis`. The name
lives in the Registry, not in the atom table, so keys can be created
and destroyed freely.

---

## Choosing between the three

| Need | Use |
|------|-----|
| Static, few, known-at-compile-time names | `Process.register` / module names |
| Dynamic keys, one node, created at runtime | `Registry` |
| Names that must resolve across nodes | `:global` (or `pg` for groups) |

## Rules of thumb

- **One Registry per purpose.** Session ids, event handlers, workers —
  separate registries, separate keyspaces, no collisions between
  concerns.
- **Register in `init/1`**, via the `name:` option where possible:
  the name exists exactly as long as the process, no stale entries.
- **Registry is per-node.** It is not a distributed name server; a key
  resolves on the node where the process lives. For cross-node
  discovery, pair it with `pg` or `:global`.
- **Lookup, then handle the miss.** A lookup can race the process
  exit; `[]` (or a dead pid) is a normal result, not an error — the
  caller treats "not there" as "go elsewhere or retry".
