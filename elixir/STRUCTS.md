---
title: "Elixir: Structs"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Structs

*Named maps with fixed keys and defaults — the idiomatic data type. Compile-time key checks, runtime pattern matching.*

---

## Definition

```elixir
defmodule Customer do
  defstruct name: "<anonymous>", address: nil, phone: nil
end

%Customer{}                                              # defaults
%Customer{name: "Sandy Claws", phone: "55554321"}        # any fields, any order
```

A struct is a map with a `__struct__` key naming its module. The
compiler enforces the field list: unknown keys fail at compile time,
so typos cannot silently survive.

---

## The guarantees

| Property | What it buys |
|----------|--------------|
| Fixed keys, defaults | Field typos fail at compile time, not in production |
| `is_map`-compatible | Every struct is a map; map functions and pattern matching work |
| No protocol dispatch on plain maps | `%Customer{}` is a `Map` but protocol implementations dispatch on the *struct* type — it is not `defimpl for: Map` |

---

## Structs vs maps

| | Struct | Map |
|---|---|---|
| Keys | Fixed by `defstruct` | Arbitrary |
| Typo behaviour | Compile-time error | Silent `nil` on access |
| Protocol dispatch | On the struct's module | On `Map` |
| Update syntax | `%{c | field: v}` (enforced) | `%{m | k: v}` |

Use a struct when the shape is a **contract** — the fields are part of
the module's API. Use a map when the shape is open — envelopes,
configuration, data passing through.

---

## The patterns that matter

- **Update, never mutate.** `%{customer | name: "New"}` returns a new
  struct; the old one is untouched (immutability is the BEAM's deal).
- **`@enforce_keys`** for fields with no sane default:

  ```elixir
  @enforce_keys [:id]
  defstruct [:id, name: nil]
  ```

  Building `%Customer{}` without `id` now fails at compile time.
- **`@derive`** for protocols: `@derive Jason.Encoder` is the standard
  serialization line.
- **Structs in event sourcing.** Domain events and commands are structs
  — the field list *is* the schema, and the enforced keys are the
  invariants. See [event-sourcing/EVENTS](../event-sourcing/EVENTS.md).

## Why it matters

A struct is a data contract the compiler checks for free. In a BEAM
system where everything is message passing, the struct is how the
shape of a message is documented and enforced at the boundary —
cheaply, at compile time, before the message ever crosses it.
