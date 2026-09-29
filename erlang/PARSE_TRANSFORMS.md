---
title: "Erlang: Parse Transforms"
layer: guide
audience: [agent, human]
stage: stable
---

# Erlang: Parse Transforms

*Erlang's metaprogramming: a module that rewrites another module's AST at compile time. Powerful, and mostly how libraries get it wrong.*

---

## What it is

A **parse transform** is a module run by the compiler *between* parsing
and code generation. It receives the forms (the abstract syntax tree) of
the module being compiled and returns a transformed list of forms:

```erlang
-module(my_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    transform(Forms).
```

The module opts in with a compile attribute:

```erlang
-compile({parse_transform, my_transform}).
```

From there the transform can inspect every function, add or remove
forms, and inject helpers — the Erlang equivalent of Elixir macros, one
level lower: Elixir macros compile down to the same abstract forms.

---

## What transforms are used for

| Use | Example |
|-----|---------|
| Code generation | `lager_transform` rewrites log calls to inject the module/line metadata |
| Wrapping calls | A transform can rewrite every remote call to add tracing or metric hooks |
| DSLs | Parsing a domain language embedded in strings, emitting forms |
| Deprecated-syntax migrations | Rewriting old constructs to new ones during upgrades |

The famous production transform is `lager`'s: it rewrites
`lager:info("...")` to pass `?MODULE` and `?LINE` implicitly — the
same ergonomics Elixir macros deliver with `quote`.

---

## The costs

- **Forms are a hostile API.** The abstract format is documented but
  verbose; transforms are fiddly to write and test.
- **Opaque compilation.** A transform can change anything about a
  module; readers of the source cannot see what runs.
- **Ordering hazards.** Multiple transforms compose in declaration
  order; interactions are a classic debugging swamp.

## Rules of thumb

1. **Reach for a macro/`-define` first.** Most metaprogramming needs in
   Erlang are served by the preprocessor; a parse transform is the
   last tool, for when you must rewrite *structure*.
2. **Keep transforms total.** A transform that fails on forms it did
   not anticipate breaks builds for everyone downstream — be
   conservative about what you rewrite.
3. **Name the dependency explicitly.** `-compile({parse_transform,
   m})` makes the magic visible at the top of the module; never apply
   transforms via build flags only.
4. **When you have Elixir, you usually do not need transforms.** Macros
   cover most of the same ground with far better ergonomics — parse
   transforms remain the Erlang-only fallback.

## Why it matters

Parse transforms explain the gap between the two languages'
metaprogramming stories: Elixir added a layer (`quote`/`unquote`) that
makes code-as-data pleasant, while Erlang works the same machinery in
raw abstract forms. Knowing the transform exists tells you what is
happening when a library rewrites your code in Erlang — and why the
advice is to avoid writing one.
