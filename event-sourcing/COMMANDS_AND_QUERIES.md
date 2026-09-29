---
title: "Event Sourcing: Commands and Queries"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Commands and Queries

*Commands change state and return nothing but status. Queries return data and change nothing. The split is CQRS in one sentence.*

---

## Command

A **command** is a message that represents an action that changes system
state. Think of it as a serialized method call: the name of the behaviour to
invoke plus its parameters.

- Named in the **imperative**: `PayFare`, `MoveItem`, `DeactivateItem`.
- Carries a message id (for deduplication) plus the parameters.
- Generally synchronous: returns **success or error** to the caller.

```elixir
%MoveItem{id: "24728347", from: "17", to: "28"}
```

**The rule people get wrong:** a command returns a status, not data. The
largest recurring mistake in CQRS systems is having commands return domain
information. Exceptions exist, but the default is status-only — the client
reads results through queries afterwards.

---

## Query

A **query** is a message that reads information without mutating business
state. `GetCustomer id=1234` → a customer DTO.

- "Does not mutate state" is a *conceptual* rule: logging the query for
  load analysis is fine; changing domain state is not.
- Usually exposed over HTTP: `GET /customers/4`. HTTP buys two things for
  free: **content-type negotiation** (JSON vs XML vs CSV per client) and
  **versioning** (a client on DTO v2 asks for v2 while others use v4).

**Versioning discipline.** Deprecate old versions over time instead of
breaking consumers; a client asking for version 3 when the system is at
version 23 needs a well-thought-out deprecation strategy, not silent
support forever.

---

## The pairing

| | Mutates state | Returns data |
|---|---|---|
| Command | yes | no (status only) |
| Query | no | yes |

If a call mutates state, it does not return domain data. If it returns
data, it does not mutate state. This invariant is what makes a CQRS split
safe to reason about: a call that returns a value can be assumed pure.

See [CQRS](CQRS.md).
