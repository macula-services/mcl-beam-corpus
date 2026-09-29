---
title: "Elixir: Macros"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Macros

*A macro is a function the compiler calls with code as data and whose returned code replaces the call. Much of Elixir itself (`if`, `def`, `use`) is built this way. Libraries need macros; application code rarely does.*

---

## Code as data

Elixir code has a plain data representation (the AST): literals stay as
they are and everything else is a three-element tuple
`{name, metadata, arguments}`. `quote` turns code into that form and
`Macro.to_string/1` turns it back:

```elixir
iex> ast = quote do: price(sku, 10)
{:price, [], [{:sku, [], Elixir}, 10]}
iex> Macro.to_string(ast)
"price(sku, 10)"
```

`unquote` inserts a value or another piece of AST into a quote, the way
string interpolation inserts into a string.

## Writing a macro

```elixir
defmodule Guarded do
  defmacro with_default(expr, default) do
    quote do
      case unquote(expr) do
        nil -> unquote(default)
        value -> value
      end
    end
  end
end

require Guarded
Guarded.with_default(Map.get(opts, :port), 4000)
```

The macro receives its arguments unevaluated (as AST) at compile time
and returns AST that the compiler expands at the call site. A macro
must be `require`d (or imported) before use, so it is always visible
where it applies. `Macro.expand_once/2` shows what a call turns into.

| Construct | Role |
|-----------|------|
| `quote` / `unquote` | build AST, splice into it |
| `defmacro` | define a compile-time function returning AST |
| `require` / `import` | make a module's macros available lexically |
| `use Mod, opts` | calls the `Mod.__using__/1` macro, which injects code into your module |
| `var!` | deliberately break hygiene to touch a caller's variable |

## Hygiene

Variables created inside a quote do not leak into, or collide with, the
caller's variables. `value` in the example above cannot overwrite a
`value` the caller already has. `var!` overrides that and should be rare.

---

## When to write one

- **Use a function if a function works.** Macros are for what functions
  cannot do: run at compile time, receive code unevaluated, generate
  definitions, or add syntax for a DSL (Ecto schemas and queries, ExUnit
  `test`, `use GenServer`).
- **Keep the quoted part small.** Put the logic in ordinary functions and
  have the macro generate calls to them; the official guide gives the
  same advice. Generated code is harder to read, debug and format.
- **Mind compile-time dependencies.** Modules that use a macro recompile
  when it changes.

Erlang's counterparts are the preprocessor and
[parse transforms](../erlang/PARSE_TRANSFORMS.md). Protocols often
remove the need for type-switching macros ([PROTOCOLS](PROTOCOLS.md)).

## Sources

- *Metaprogramming Elixir: Write Less Code, Get More Done (and Have Fun!)*, 1st edition, Chris McCord, Pragmatic Bookshelf, 2015. <https://pragprog.com/titles/cmelixir/metaprogramming-elixir/>
- Elixir guide, Quote and unquote. <https://elixir.hexdocs.pm/quote-and-unquote.html>
- Elixir guide, Macros. <https://elixir.hexdocs.pm/macros.html>
- Elixir `Macro` documentation. <https://elixir.hexdocs.pm/Macro.html>
