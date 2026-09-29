---
title: "Erlang: Parse Transforms"
layer: guide
audience: [agent, human]
stage: stable
---

# Erlang: Parse Transforms

*A parse transform is a module the compiler calls to rewrite another module's syntax tree before checking and code generation. It is Erlang's most powerful metaprogramming tool and the one OTP advises against.*

---

## What it is

When the compiler has parsed a module into its **abstract format** (a
list of forms: attributes and function definitions as nested tuples), it
passes that list to each requested parse transform. A transform is any
module exporting `parse_transform/2`:

```erlang
-module(stamp_transform).
-export([parse_transform/2]).

%% Add an exported build_info/0 returning the compile time.
parse_transform(Forms, _Options) ->
    Stamp = calendar:system_time_to_rfc3339(erlang:system_time(second)),
    Fun = {function, 0, build_info, 0,
           [{clause, 0, [], [], [erl_parse:abstract(Stamp)]}]},
    {Body, [Eof]} = lists:split(length(Forms) - 1, Forms),   % last form is {eof, _}
    lists:flatmap(fun add_export/1, Body) ++ [Fun, Eof].

add_export({attribute, _, module, _} = M) -> [M, {attribute, 0, export, [{build_info, 0}]}];
add_export(F) -> [F].
```

A module opts in with `-compile({parse_transform, stamp_transform}).`
(or the compiler option of the same name). The transform runs before
the code is checked for errors, so it can accept code that would not
compile on its own and turn it into code that does.
`erl_id_trans` in STDLIB is the reference identity transform to start
from.

---

## What people use them for

| Use | Example |
|-----|---------|
| Implicit call-site metadata | logging libraries (historically `lager`) rewriting log calls to add module, function and line |
| Compile-time generation | adding functions derived from records or attributes |
| Query DSLs | `ms_transform`, shipped with OTP, turns `ets:fun2ms(fun(...) -> ... end)` into a match specification |
| Instrumentation | wrapping calls for tracing or metrics |

---

## Costs

- **OTP's own warning:** the `erl_id_trans` docs say programmers are
  "strongly advised not to engage in parse transformations" and that no
  support is offered for problems encountered.
- **Invisible semantics.** The source no longer says what runs; readers,
  tools and debuggers see different code.
- **Fragile coupling to the abstract format,** which gains new node types
  as the language evolves; a transform that does not handle them breaks
  its users' builds.
- **Ordering.** Several transforms are applied in sequence and can
  interfere with each other.

## Rules of thumb

1. **Try the preprocessor first.** `-define` macros with `?MODULE`,
   `?FUNCTION_NAME` and `?LINE` cover most call-site metadata needs
   (OTP's `logger` macros do exactly this).
2. **Pass through what you do not recognise,** unchanged, so new syntax
   does not break the transform.
3. **Make the dependency visible** with a `-compile` attribute in the
   module, not only a build-tool flag.
4. **In Elixir, use macros instead.** They work on Elixir's AST with
   hygiene and explicit `require` ([MACROS](../elixir/MACROS.md)).

Related: [MODULES_AND_RECORDS](MODULES_AND_RECORDS.md).

## Sources

- *Erlang and OTP in Action*, 1st edition, Martin Logan, Eric Merritt and Richard Carlsson, Manning, 2010. <https://www.manning.com/books/erlang-and-otp-in-action>
- Compiler `compile` reference (`{parse_transform, Module}` option). <https://www.erlang.org/doc/apps/compiler/compile.html>
- STDLIB `erl_id_trans` reference (identity transform and warning). <https://www.erlang.org/doc/apps/stdlib/erl_id_trans.html>
- ERTS, The Abstract Format. <https://www.erlang.org/doc/apps/erts/absform.html>
- Erlang/OTP Reference Manual, Preprocessor (predefined macros). <https://www.erlang.org/doc/system/macros.html>
