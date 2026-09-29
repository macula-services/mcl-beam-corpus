---
title: Elixir
layer: index
audience: [agent, human]
stage: stable
---

# Elixir

*The language: macros, protocols, structs, mix.*

Elixir runs on the BEAM and inherits its concurrency and fault-tolerance
model; this domain holds what is Elixir-specific on top of that.

---

## Notes

| Note | Covers |
|------|--------|
| [MACROS](MACROS.md) | Quote/unquote, `use`, hygiene, macros are for libraries |
| [PROTOCOLS](PROTOCOLS.md) | Dispatch by first argument's type, `@derive`, consolidation |
| [STRUCTS](STRUCTS.md) | Fixed keys, enforced keys, structs vs maps |
| [MIX](MIX.md) | Project layout, the tasks worth knowing, releases |

## One-page summary

Elixir compiles to BEAM bytecode. Its distinctive features over Erlang:

- **Macros** run at compile time and return AST, giving libraries the
  same expressive power as the language core (`if` and `defmodule` are
  macros).
- **Protocols** provide polymorphism dispatched on the first argument's
  type, implemented per type, consolidated at build time for speed.
- **Structs** are named maps with fixed keys and defaults — the idiomatic
  data type.
- **mix** is the build tool: `mix new`, `mix deps.get`, `mix test`,
  `mix release`.

The one rule that matters everywhere else: metaprogramming is for
libraries, not for application code. Write macros only when plain
functions cannot express the thing.
