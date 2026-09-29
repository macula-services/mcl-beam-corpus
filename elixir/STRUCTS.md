---
title: "Elixir: Structs"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Structs

*Maps with a fixed set of keys, defaults and a type name. The idiomatic way to give data a shape the compiler can check.*

---

## Definition

```elixir
defmodule Station do
  @enforce_keys [:id]
  defstruct [:id, region: "unassigned", peers: []]
end

%Station{id: "st-07"}                      # region and peers take defaults
%Station{id: "st-07", region: "eu-west"}   # any fields, any order
```

A struct is a map with a `__struct__` key holding its module name. When
the struct is known at compile time, building it with an unknown key is
a compile-time `KeyError`, and leaving out an `@enforce_keys` field is a
compile-time `ArgumentError`.

---

## Structs versus maps

| | Struct | Map |
|---|---|---|
| Keys | fixed by `defstruct` | arbitrary |
| Unknown key when building | error (compile time for literals) | allowed |
| `data.key` on a missing key | `KeyError` | `KeyError` |
| `data[:key]` | not supported: structs do not implement `Access` | returns `nil` if missing |
| Update `%{s | k: v}` | key must exist | key must exist |
| Protocols | dispatch on the struct module; no `Map` implementations inherited | dispatch on `Map` |
| `is_map/1` | `true` | `true` |

Use a struct when the shape is a **contract**: the fields are part of the
module's API. Use a map when the shape is open: envelopes, configuration,
data passing through. Because structs do not inherit map protocols,
`Enum` functions do not work on a struct unless it implements
`Enumerable`.

---

## Patterns that matter

- **Update, never mutate.** `%{station | region: "eu-north"}` returns a new
  struct; the old value is unchanged.
- **`@enforce_keys`** for fields without a sensible default. It is checked
  when the struct is built, not when it is updated.
- **Pattern-match on the type** in function heads: `def link(%Station{} = s)`
  documents and enforces the argument's shape.
- **`@derive`** for protocols such as `Inspect` (hide fields) or
  `JSON.Encoder` ([PROTOCOLS](PROTOCOLS.md)).
- **Commands and events as structs.** In event-sourced code the field list
  is the schema and enforced keys are its required fields; see
  [event-sourcing/EVENTS](../event-sourcing/EVENTS.md).

## Relation to Erlang records

Erlang's records are compile-time names over tuples
([MODULES_AND_RECORDS](../erlang/MODULES_AND_RECORDS.md)); structs are
maps that carry their type at runtime, so another module can match on
them or inspect them without including a shared header file.
Elixir's `Record` module exists for interoperating with Erlang records.

## Sources

- Elixir guide, Structs. <https://elixir.hexdocs.pm/structs.html>
- Elixir `Kernel` documentation (`defstruct/1`). <https://elixir.hexdocs.pm/Kernel.html>
- Elixir `Record` documentation. <https://elixir.hexdocs.pm/Record.html>
