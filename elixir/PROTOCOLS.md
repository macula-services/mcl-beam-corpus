---
title: "Elixir: Protocols"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: Protocols

*Polymorphism dispatched on the type of the first argument. Anyone can add an implementation for any type; Mix consolidates them at build time for fast dispatch.*

---

## The shape

```elixir
defprotocol Describe do
  @doc "A short human-readable label for a value"
  def label(value)
end

defimpl Describe, for: Integer do
  def label(n), do: "#{n} (integer)"
end

defimpl Describe, for: Station do          # a struct defined elsewhere
  def label(%Station{id: id, region: r}), do: "station #{id} in #{r}"
end

Describe.label(42)                          #=> "42 (integer)"
```

A protocol declares functions; implementations provide them per data
type. The first argument's type selects the implementation at call time:
built-in types (`Integer`, `List`, `Map`, `BitString`, ...) or a struct
module.

---

## What protocols solve

Behaviours give polymorphism on the **module**: the caller chooses which
module to call. Protocols give polymorphism on the **data**: the caller
passes a value and the right implementation is found from its type. That
fits operations such as "render as text", "encode as JSON", "enumerate",
"inspect".

Implementations can live with the protocol, with the type, or in a third
library, so a package can make its structs work with a protocol it does
not own, or implement its protocol for types it does not own.

---

## The machinery

| Piece | Role |
|-------|------|
| `@fallback_to_any true` | Types without an implementation use the `Any` implementation instead of raising `Protocol.UndefinedError` |
| `@derive [Proto]` | On a struct, reuse the protocol's `Any` implementation (which may customise itself per struct, as `Inspect` and `JSON.Encoder` do) |
| Consolidation | Mix, by default during compilation, rewrites each protocol so dispatch goes straight to known implementations. Unconsolidated dispatch must look implementations up at runtime and is slower. Set `consolidate_protocols: false` when tests must define implementations dynamically. |

---

## Built-ins to know

`Enumerable` (behind `Enum`), `Collectable` (behind `Enum.into` and
`for ... into:`), `Inspect`, `String.Chars` (`to_string/1` and
interpolation), `List.Chars`, and `JSON.Encoder` (Elixir 1.18+; the
Jason library's `Jason.Encoder` plays the same role). Deriving an encoder
on a struct is the everyday case.

## Rules of thumb

- **Put the varying type first.** Dispatch only looks at the first
  argument.
- **Prefer an existing protocol** (`String.Chars`, `Inspect`) to a new
  one with the same meaning.
- **Do not fall back to `Any` casually.** A missing implementation that
  raises is usually a bug worth seeing.
- **Case-ing on types in application code** is often a protocol asking
  to be written.

Structs get their own dispatch type, separate from `Map`
([STRUCTS](STRUCTS.md)). Protocols are also how the Visitor pattern
disappears on the BEAM ([DESIGN_PATTERNS](../beam/DESIGN_PATTERNS.md)).

## Sources

- Elixir guide, Protocols. <https://elixir.hexdocs.pm/protocols.html>
- Elixir `Protocol` documentation (consolidation, `@fallback_to_any`, deriving). <https://elixir.hexdocs.pm/Protocol.html>
