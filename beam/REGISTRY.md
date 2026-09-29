---
title: "BEAM: Registry"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Registry

*Elixir's local key-to-process directory: name processes with any term, find them without going through a single server.*

---

## The problem it solves

A pid is only good while its process lives; after a restart the
supervisor hands out a new one. So callers need a stable name. The BEAM
offers several naming mechanisms:

| Mechanism | Keys | Scope | Limitation |
|-----------|------|-------|------------|
| `Process.register/2` / `name: MyMod` | atoms | one node | atoms are never garbage collected, so they must not be minted from runtime data |
| `:global` | any term | whole cluster | cluster-wide locking on registration; costly at scale |
| `Registry` (Elixir) | any term | one node | local only |
| `:pg` | any term (group names) | cluster | groups, not unique names; eventually consistent |

`Registry` is the right default for "one process per user, session,
device or job" on a single node.

---

## How it works

A registry is started under your supervision tree and is backed by ETS
tables, optionally split into partitions for concurrency. A process
registers **itself** (`Registry.register/3` always acts on the calling
process). When a registered process exits, its entries are removed
automatically; the docs note the removal may not be visible
immediately.

```elixir
# in the application's children list
{Registry, keys: :unique, name: Devices.Registry}

# name a GenServer by a runtime key
GenServer.start_link(Devices.Link, serial,
  name: {:via, Registry, {Devices.Registry, serial}})

# address it later by the same key
GenServer.call({:via, Registry, {Devices.Registry, "A7-1182"}}, :status)
Registry.lookup(Devices.Registry, "A7-1182")   #=> [{pid, value}] or []
```

| `keys:` | Meaning | Typical use |
|---------|---------|-------------|
| `:unique` | at most one process per key | naming workers |
| `:duplicate` | many processes per key | local pub/sub, dispatch to all subscribers |

The value stored at registration (also settable through the three-element
via tuple `{:via, Registry, {reg, key, value}}`) comes back with every
lookup.

---

## Pitfalls

- **Local only.** A key resolves on the node that holds the registry.
  Across nodes use `:pg`, `:global`, or a design that routes to the
  owning node.
- **A lookup is a snapshot.** The process can exit right after you get
  its pid. Treat `[]` and a dead pid as ordinary outcomes.
- **Register through `name:` at start**, so the name exists exactly as
  long as the process and restarts re-register automatically.
- **Separate registries per concern** keep keyspaces apart and make
  intent obvious.

Related: [GENSERVER](GENSERVER.md), [SUPERVISION_TREES](SUPERVISION_TREES.md), [ETS](ETS.md), [DISTRIBUTION](DISTRIBUTION.md).

## Sources

- *Designing Elixir Systems with OTP*, 1st edition, James Edward Gray II and Bruce A. Tate, Pragmatic Bookshelf, 2019. <https://pragprog.com/titles/jgotp/designing-elixir-systems-with-otp/>
- Elixir `Registry` documentation. <https://elixir.hexdocs.pm/Registry.html>
- Erlang/OTP `pg` reference. <https://www.erlang.org/doc/apps/kernel/pg.html>
