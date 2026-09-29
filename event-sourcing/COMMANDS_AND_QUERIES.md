---
title: "Event Sourcing: Commands and Queries"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Commands and Queries

*A command asks the system to change and answers only "done" or "refused". A query answers a question and changes nothing. Keeping the two apart is the idea CQRS is built on.*

---

## The principle underneath

Bertrand Meyer's command-query separation (CQS) says a method should
either change state or return information, not both. CQRS applies the
same split to whole messages and, eventually, to whole models
(see [CQRS](CQRS.md)).

## Commands

A command is a request to perform a business action. It is named in the
imperative, carries the data the action needs, and has a unique id so
that a retried delivery can be recognised:

```elixir
%ReserveSeat{command_id: "c7f1…", show_id: "2026-10-03-evening", seat_id: "B-12", customer_id: "cust-981"}
```

What a command handler gives back is an outcome, not a view of the
domain:

```elixir
:ok
{:error, :seat_already_taken}
```

The caller learns *whether* it worked. If it needs to see the result, it
asks a query afterwards (or reads the events the command produced).
Returning domain data from commands is the most common way the split
erodes: soon clients depend on the write model's shape and the read side
can no longer evolve on its own. There are pragmatic exceptions, such as
returning the id or new stream version of what was just written, but
treat them as exceptions.

A command may be refused. Zero events is a valid, ordinary outcome.

## Queries

A query asks for information and must not change business state:

```elixir
%SeatsAvailable{show_id: "2026-10-03-evening"}
# => [%{seat_id: "A-01", price_cents: 2400}, …]
```

"Must not change state" is about the domain. Logging, metrics and caches
touched while answering are fine.

Queries are served from read models built for them
([PROJECTIONS](PROJECTIONS.md)), never by folding the event store on the
request path. Because readers differ, the read side is also where you
offer several representations (JSON, CSV) and several versions of a
response. Retire old versions deliberately, with a published deprecation
window, rather than supporting every version forever or breaking
clients without warning.

## Side by side

| | Changes business state | Returns domain data | May be refused |
|---|---|---|---|
| Command | yes | no, an outcome only | yes |
| Query | no | yes | only for access or validation reasons |

## On the BEAM

A natural shape is `GenServer.call/3` to the aggregate process for a
command, returning `:ok | {:error, reason}`, and a direct read of an ETS
table or SQL read model for a query, which never goes through the
aggregate process at all. That keeps slow or heavy reads from queueing
behind writes in a process mailbox.

## Sources

- Greg Young, *CQRS Documents*, self-published PDF, 2010 (free). https://cqrs.wordpress.com/wp-content/uploads/2010/11/cqrs_documents.pdf
- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Martin Fowler, "Command Query Separation", martinfowler.com, 2005 (free). https://martinfowler.com/bliki/CommandQuerySeparation.html
- Microsoft, "CQRS pattern", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
