---
title: Erlang
layer: index
audience: [agent, human]
stage: stable
---

# Erlang

*The language: modules, records, parse transforms.*

Erlang is the original BEAM language; Elixir's semantics are Erlang's.
This domain holds what is Erlang-specific, and what Elixir code calls
down into (OTP modules, ETS, the `:erlang` BIFs).

---

## Notes

None yet. See [Planned](#planned).

## Planned

- Modules and exports: compile vs runtime, `-on_load`, code paths
- Records: compile-time tuples, in header files
- Parse transforms: compile-time AST rewrites (Erlang's macros)
- The `:erlang` module: BIFs worth knowing (`:erlang.phash2`, `monitor`,
  `spawn_opt`, `:erlang.term_to_binary`)

## One-page summary

Erlang syntax is Prolog-derived: `-module(foo).` attributes, function
clauses with `;` separators, single assignment. Everything Elixir does
compiles to these constructs. Records are compile-time tuples; parse
transforms are the metaprogramming layer. When an Elixir library says
"OTP", it means the Erlang modules (`:gen_server`, `:ets`, `:supervisor`)
called directly.
