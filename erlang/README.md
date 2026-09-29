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

| Note | Covers |
|------|--------|
| [MODULES_AND_RECORDS](MODULES_AND_RECORDS.md) | Exports, attributes, records as tagged tuples, header files |
| [PARSE_TRANSFORMS](PARSE_TRANSFORMS.md) | Compile-time AST rewrites, the Erlang metaprogramming fallback |

## One-page summary

Erlang syntax is Prolog-derived: `-module(foo).` attributes, function
clauses with `;` separators, single assignment. Everything Elixir does
compiles to these constructs. Records are compile-time tuples with
names; parse transforms are the metaprogramming layer. When an Elixir
library says "OTP", it means the Erlang modules (`:gen_server`, `:ets`,
`:supervisor`) called directly.
