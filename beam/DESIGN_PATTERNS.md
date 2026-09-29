---
title: "BEAM: The GoF Patterns on the BEAM"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: The GoF Patterns on the BEAM

*The classic 23 still apply — but the BEAM replaces some with built-ins, rebrands others, and turns a few into anti-patterns. A mapping.*

---

## Still patterns — implement as modules

| GoF | On the BEAM |
|-----|-------------|
| Adapter | A plain module translating one interface to another |
| Facade | A plain module hiding machinery (the GenServer API shape) |
| Decorator | Function composition and pipelines; `|>` chains |
| Strategy | Higher-order functions, or a behaviour with several implementations |

Nothing changes for these — they are about module boundaries, and the
BEAM has modules.

---

## Replaced by the language and runtime

| GoF | The BEAM answer |
|-----|-----------------|
| Singleton | **Anti-pattern as mutable global.** Application env for config; a registered process ([REGISTRY](REGISTRY.md)) or a named GenServer for one stateful instance |
| Observer | Process monitoring, subscriptions, `Registry` with `keys: :duplicate`, `Phoenix.PubSub` — pub/sub is native |
| Iterator | `Enum` and `Stream` — iteration is data, not objects |
| State | `gen_statem` / a GenServer's state transitions |
| Template method | Behaviours: the callback list *is* the template |
| Chain of responsibility | Function pipelines; `Plug` |
| Memento | `:erlang.term_to_binary/1` — and in event-sourced systems the event log *is* the memento; no separate snapshot needed |
| Prototype | Rarely needed — immutability makes copying trivial |
| Flyweight | ETS tables, atom interning, refcounted binaries — sharing is the runtime's business |
| Proxy | The process boundary: a GenServer's API module is a natural proxy |
| Visitor | **Protocols** — dispatch on the visited type, per type implementations ([PROTOCOLS](../elixir/PROTOCOLS.md)) |
| Interpreter | Macros and DSLs when truly needed — [MACROS](../elixir/MACROS.md) |
| Mediator | Event channels, a broker — see [EVENT_DRIVEN_ARCHITECTURE](../architecture/EVENT_DRIVEN_ARCHITECTURE.md) |

---

## The name collision that bites

**Command** means two things:

| Context | Meaning |
|---------|---------|
| GoF | Encapsulate a request as an object (undo, queues) |
| CQRS / event sourcing | A message that *changes state*, returning nothing but status |

They are different patterns sharing a word. On the BEAM, the GoF
command is usually just a message (`GenServer.cast`); the CQRS command
is the domain concept — see
[COMMANDS_AND_QUERIES](../event-sourcing/COMMANDS_AND_QUERIES.md).

---

## Factory patterns, deflated

Abstract Factory, Factory Method, Builder carry ceremony the BEAM
does not need:

- Factory Method → a plain constructor function.
- Builder → a struct with defaults, keyword options, or a changeset.
- Abstract Factory → a behaviour chosen by config.

Write the function; skip the machinery.

## Rules of thumb

- When you reach for a GoF pattern, first ask what the **runtime
  already does**: processes, monitors, ETS, and protocols cover most
  of the catalogue.
- A pattern implemented where a built-in exists is code the next
  reader must learn — the built-in is already known.
- The surviving patterns are the module-boundary ones (Adapter,
  Facade, Strategy). The rest are features.
