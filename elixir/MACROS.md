---
title: "Elixir: Macros"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Macros

*Macros run at compile time and return AST. Elixir itself is built with them — `if` and `defmodule` are macros. That power is for libraries, not application code.*

---

## Quote and unquote

`quote` captures code as its AST representation:

```elixir
iex> quote do: 1 + 2
{:+, [context: Elixir, imports: [{1, Kernel}, {2, Kernel}]], [1, 2]}
```

A macro is a function that receives quoted arguments and returns quoted
code, which the compiler substitutes at the call site. `unquote`
splices evaluated values into that AST:

```elixir
defmacro times(lhs, rhs) do
  quote do
    lhs = unquote(lhs)   # the caller's expression, spliced in
    rhs = unquote(rhs)
    lhs * rhs
  end
end
```

---

## How the pieces fit

| Piece | What it does |
|-------|--------------|
| `quote do: ...` | Code → AST |
| `unquote(x)` | Value → AST, spliced into the surrounding quote |
| `defmacro` | Defines a compile-time function returning AST |
| `use Mod` | Expands `Mod.__using__/1` — a macro — into the caller |

`use` is the dominant macro pattern: a library defines `__using__`, and
`use GenServer` injects the callback scaffolding into the calling
module.

---

## Macro hygiene

Macros are hygienic by default: variables they introduce do not clash
with the caller's variables, and imported functions resolve in the
macro's context. `var!` opts out of hygiene deliberately — the escape
hatch you almost never need.

---

## The rules

1. **Macros are for libraries.** Application code should read as plain
   functions; metaprogramming is how libraries earn their ergonomics
   (Ecto's `schema`, Phoenix's `plug`, `use GenServer`).
2. **Prefer functions until they cannot express the thing.** A macro
   exists to do what a function cannot: run at compile time, inject
   code, or extend the language. If a function would work, use the
   function.
3. **Return plain AST.** Keep quoted code boring; the debugger,
   formatter, and reader all have to handle what the macro produces.
4. **One level of expansion.** `use` calls `__using__` which calls other
   macros — keep the chain shallow and predictable.

## Why it matters

Macros are why Elixir's core can be small and its libraries expressive.
But every macro is a language extension, and every extension is a
liability for whoever reads the code next. The discipline is the value:
write macros only where they pay for their cost in clarity.
