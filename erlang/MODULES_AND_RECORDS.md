---
title: "Erlang: Modules and Records"
layer: guide
audience: [agent, human]
stage: stable
---

# Erlang: Modules and Records

*Modules are the unit of code, loading and visibility; records are compile-time field names over tuples. Most Erlang and OTP code is written with both.*

---

## Modules

A module is one source file (`meter.erl` defines `meter`) made of
attributes followed by functions. Only exported functions can be called
from outside, and a function is identified by name **and** arity:
`read/1` and `read/2` are unrelated functions.

```erlang
-module(meter).
-export([new/1, record/2, total/1]).

new(Id) -> {meter, Id, 0}.
record({meter, Id, T}, Kwh) when Kwh >= 0 -> {meter, Id, T + Kwh}.
total({meter, _Id, T}) -> T.
```

| Attribute | Purpose |
|-----------|---------|
| `-export([F/A, ...]).` | the public API |
| `-import(Mod, [F/A, ...]).` | call another module's functions without the prefix (hurts readability; use rarely) |
| `-behaviour(gen_server).` | declare which callbacks the module implements; the compiler warns about missing ones |
| `-compile(Options).` | per-module compiler options, e.g. a parse transform |
| `-on_load(F/0).` | function run automatically when the module is loaded |
| `-include("file.hrl").` / `-include_lib(...)` | textual inclusion of headers (records, macros) |

Modules are also the unit of code loading: the runtime can hold a
current and an old version of each one, which is the basis of
[hot code upgrades](../beam/HOT_CODE_UPGRADES.md).

---

## Records

Tuples are the compact, fast way to group values, but positional access
breaks every pattern when a field is added. A record gives the positions
names at compile time:

```erlang
-record(reading, {sensor, value = 0.0, unit = celsius, taken_at}).

R  = #reading{sensor = <<"t-12">>, taken_at = erlang:system_time(second)},
#reading{value = V} = R,          % match one field
Unit = R#reading.unit,            % access
R2 = R#reading{value = 21.5}.     % copy with a changed field
```

At runtime `R` is just `{reading, <<"t-12">>, 0.0, celsius, 1759000000}`:
the record name, then the fields in declaration order. Fields not given
a value take their default, or `undefined` if none was declared.
`record_info(fields, reading)` lists the field names at compile time.

### Sharing records

The compiler must see a record's definition in every module that uses
it, so shared records live in `.hrl` header files included where
needed. OTP itself ships records this way, for example `#file_info{}` in
Kernel's `file.hrl`.

---

## Trade-offs

- **Records cost nothing at runtime** and give pattern matching on named
  fields, but they couple every module that includes the header: change
  the record and all of them must be recompiled together. Data stored or
  sent between nodes in the old shape does not update itself.
- **Maps** are self-describing and flexible, at slightly higher cost;
  they suit open-ended data and data that crosses version boundaries.
- Elixir's answer is the **struct**, a map that carries its type
  ([STRUCTS](../elixir/STRUCTS.md)); Elixir's `Record` module exists for
  working with Erlang records.

## Sources

- *Erlang and OTP in Action*, 1st edition, Martin Logan, Eric Merritt and Richard Carlsson, Manning, 2010. <https://www.manning.com/books/erlang-and-otp-in-action>
- Erlang/OTP Reference Manual, Modules. <https://www.erlang.org/doc/system/modules.html>
- Erlang/OTP Reference Manual, Records. <https://www.erlang.org/doc/system/ref_man_records.html>
- Erlang/OTP Reference Manual, Preprocessor (include, macros). <https://www.erlang.org/doc/system/macros.html>
