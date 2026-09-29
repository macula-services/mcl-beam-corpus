---
title: "BEAM: The GoF Patterns on the BEAM"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: The GoF Patterns on the BEAM

*The classic object-oriented pattern catalogue, read from a functional, process-based runtime: some patterns survive as module design, many become language or runtime features, a few turn into anti-patterns.*

---

## Why a mapping is needed

The 1994 catalogue solves problems of class-based languages: how to vary
behaviour without subclass explosions, how to share objects safely, how
to decouple senders from receivers. The BEAM has no classes or mutable
objects, but it has modules, higher-order functions, behaviours,
protocols and processes. Many of the catalogue's problems are therefore
already solved, and writing the pattern out by hand adds code a reader
must decode.

---

## Patterns that survive as module design

| Pattern | BEAM form |
|---------|-----------|
| Adapter | a module translating one API into another |
| Facade | one public module in front of a library's internals; also the client API of a [GenServer](GENSERVER.md) |
| Strategy | pass a function, or pick a module implementing a behaviour |
| Decorator | wrap a function in another; compose with `|>` |

## Patterns absorbed by the language or runtime

| Pattern | What replaces it |
|---------|------------------|
| Template Method | a behaviour: the callback list is the template, `use` can inject defaults |
| Visitor | a [protocol](../elixir/PROTOCOLS.md): per-type implementations, dispatch on the data |
| Iterator | `Enum` and lazy `Stream` over any `Enumerable` |
| Observer | monitors, `Registry` with duplicate keys, `:pg`, Phoenix.PubSub |
| State | `gen_statem`, or a GenServer whose state carries the current mode |
| Chain of Responsibility | a list of functions or plugs applied in order |
| Proxy | the process boundary: the API module forwards to a process that may be local or remote |
| Flyweight | runtime sharing: atoms, reference-counted large binaries, [ETS](ETS.md) |
| Prototype | immutability: "copying" is just reusing the value |
| Memento | immutable values are snapshots already; in event-sourced systems the event log is the history, with snapshots as an optimisation |
| Mediator | a broker process or event channel; see [EVENT_DRIVEN_ARCHITECTURE](../architecture/EVENT_DRIVEN_ARCHITECTURE.md) |
| Interpreter | pattern matching over a data structure; [macros](../elixir/MACROS.md) only when compile-time syntax is truly needed |

## Creational patterns shrink to functions

Factory Method becomes a constructor function (`new/1`). Builder becomes
a struct with defaults plus keyword options or a changeset. Abstract
Factory becomes a behaviour whose implementation is chosen by
configuration.

## Singleton is an anti-pattern

A mutable global does not exist on the BEAM, and imitating one creates a
bottleneck. Configuration goes in the application environment; a single
stateful thing is a named process under supervision, with the
consequences that implies (serialised access, restarts).

## One word, two patterns: Command

In the catalogue, a command is a request packaged as an object so it can
be queued or undone. In CQRS and event sourcing, a command is an
intention to change a domain aggregate, validated and answered with
events or a rejection. On the BEAM the first is usually just a message;
the second is a domain concept, see
[COMMANDS_AND_QUERIES](../event-sourcing/COMMANDS_AND_QUERIES.md).

## Rule of thumb

Before implementing a named pattern, ask what processes, behaviours,
protocols, monitors and ETS already give you. A built-in is known to
every reader; a hand-rolled pattern is not.

## Sources

- *Design Patterns: Elements of Reusable Object-Oriented Software*, Erich Gamma, Richard Helm, Ralph Johnson and John Vlissides, Addison-Wesley, 1994. <https://www.informit.com/store/design-patterns-elements-of-reusable-object-oriented-9780201633610>
- Elixir guide, Protocols. <https://elixir.hexdocs.pm/protocols.html>
- Elixir documentation, Typespecs and behaviours. <https://elixir.hexdocs.pm/typespecs.html>
- Erlang/OTP `gen_statem` reference. <https://www.erlang.org/doc/apps/stdlib/gen_statem.html>
