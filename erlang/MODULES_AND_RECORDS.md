---
title: "Erlang: Modules and Records"
layer: guide
audience: [agent, human]
stage: stable
---

# Erlang: Modules and Records

*Modules group functions; records name tuple fields at compile time. The two constructs everything else in Erlang builds on.*

---

## Modules

A module is a file of function clauses, declared with attributes:

```erlang
-module(simple_cache).
-export([insert/2, lookup/1, delete/1]).

insert(Key, Value) -> ... .

lookup(Key) -> ... .

delete(Key) -> ... .
```

Only exported functions are callable from outside. A function is named
by `name/arity` — `insert/2` and `insert/3` are *different functions*
that happen to share a name.

The attributes that matter:

| Attribute | Role |
|-----------|------|
| `-export/1` | The public API |
| `-import/1` | Bring functions in unqualified (use sparingly) |
| `-behaviour/1` | Declare a callback contract (checked by the compiler) |
| `-compile/1` | Compiler flags (`export_all`, warnings) |

---

## Records — named tuple fields

Erlang's structured data type is the **tuple**, fast and compact — but
add a field and every pattern in the codebase breaks. Records solve
that: compile-time names over fixed-shape tuples.

```erlang
-record(customer, {name = "<anonymous>", address, phone}).
```

This declares a **tagged tuple** of four elements (three fields plus the
tag `customer`), with field order fixed by the declaration. Creating
and matching:

```erlang
C = #customer{name = "Sandy Claws", phone = "55554321"},  %% fields in any order
#customer{name = Name, phone = Phone} = C,                %% match
C#customer.name                                           %% access
C#customer{name = "New Name"}                             %% "update" (new tuple)
```

Fields left unset get their declared default, or the atom `undefined`.

### Where declarations live

Records are **compile-time**, not runtime types: the declaration must
be visible to the compiler of every module that uses it. Convention:
declare shared records in `.hrl` header files and include them:

```erlang
-include("customer.hrl").
```

---

## The trade

Records exist because tuples are the fastest, smallest structured data
on the BEAM — records give tuples names without costing anything at
runtime. The cost is compile-time coupling: every module sharing a
record depends on the header, and the header must change together with
every consumer. That is the accepted price of the performance; when
flexibility matters more, maps are the alternative.

## Why it matters

All OTP data the runtime hands you — `#child_spec{}`, application
resource files, observer records — is records. Reading Erlang and OTP
means reading records; knowing they are compile-time names over tuples
is the whole explanation of their speed, their header-file coupling,
and why Elixir replaced them with structs on its side.
