---
title: "Elixir: Protocols"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Protocols

*Polymorphism dispatched on the first argument's type. Extendable by anyone, consolidated at build time for speed.*

---

## The shape

```elixir
defprotocol Blank do
  @fallback_to_any true
  def blank?(data)
end

defimpl Blank, for: List do
  def blank?([]), do: true
  def blank?(_),  do: false
end

defimpl Blank, for: Map do
  def blank?(map), do: map_size(map) == 0
end
```

A protocol declares functions; implementations provide them **per type**.
Dispatch happens on the type of the first argument — `blank?([])` finds
the `List` implementation at runtime.

---

## What protocols solve

Behaviours give polymorphism on the **module** (one implementation per
module). Protocols give polymorphism on the **data**: one function name,
many behaviours by data type, chosen at dispatch. That is the exact
case for "to string", "to JSON", "size of", "enumerable" — operations
whose implementation depends on what the value *is*.

The library payoff: any package can implement `MyProtocol` for types it
does not own, without touching either the protocol or the type's own
module.

---

## The machinery

| Piece | Role |
|-------|------|
| `@fallback_to_any true` | Unimplemented types fall back to `Any` (default: raises `Protocol.UndefinedError`) |
| `@derive Protocol` | Generate the implementation from the struct's fields (`@derive Jason.Encoder`) |
| **Consolidation** | At build time, the runtime type→implementation map is precomputed into the protocol module — turning O(types) dispatch into a table lookup. `mix release` consolidates automatically; unconsolidated protocols are slower and log a warning |

---

## Built-ins to know

`Enumerable` (the `Enum` contract), `Collectable`, `Inspect`,
`String.Chars` (`to_string`), `Jason.Encoder`/`JSON.Encoder` — the
last two are the everyday case: `@derive Jason.Encoder` on a struct
makes it serializable with one line.

## Rules of thumb

- **First argument dispatches.** Design protocols so the type that
  varies is the first argument — everything else is a parameter.
- **Consolidate.** `mix release` does it; remember that consolidation
  freezes the protocol, so test unconsolidated in dev and know the
  difference.
- **A protocol per concept, not per case.** `String.Chars` covers
  everything that renders as text; do not build `WidgetFormatter` when
  `to_string` will do.
- **Prefer structs + `@derive`** over hand-written `defimpl` for the
  serialization case — one line, and it survives field additions
  automatically.

## Why it matters

Protocols are the abstraction that lets Elixir libraries interoperate
without a central registry: the protocol is the contract, the
implementations are per type, and dispatch is a consolidated table
lookup. When you find yourself case-ing on types in application code, a
protocol is usually what the language is asking for.
