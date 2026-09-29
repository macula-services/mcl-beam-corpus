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

None yet. See [Planned](#planned).

## Planned

- Macros: quote/unquote, compile-time code generation, when not to macro
- Protocols: dispatch by type, deriving, consolidation
- Structs vs maps: defaults, compile-time key checks, enforcing keys
- mix tasks: project layout, deps, releases, `mix xref`

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
