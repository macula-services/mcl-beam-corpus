---
title: mcl-beam-corpus — Index
layer: index
audience: [agent, human]
stage: stable
---

# mcl-beam-corpus

*Knowledge corpus for BEAM languages (Erlang, Elixir), OTP, and Event Sourcing / CQRS.*

This index maps every note in the corpus. An agent answering a question
recalls from this repo through mcl-rag; this file is the human-readable
map of what is here and where a fuller answer lives.

---

## Domain map

### BEAM & OTP — `beam/`

The virtual machine and the fault-tolerance model every service on the
mesh is built on.

| Note | Covers |
|------|--------|
| [README](beam/README.md) | The BEAM in one page: processes, scheduling, memory |
| [SUPERVISION_TREES](beam/SUPERVISION_TREES.md) | Failure model, supervisor strategies, child specs, DynamicSupervisor |
| [APPLICATIONS](beam/APPLICATIONS.md) | The composition unit: resource files, start types, dependencies |
| [ETS](beam/ETS.md) | In-memory tables: types, ownership, what it is and is not for |
| [SCHEDULER](beam/SCHEDULER.md) | The m:n model, reductions, concurrency vs parallelism |
| [DISTRIBUTION](beam/DISTRIBUTION.md) | Location transparency, nodes, EPMD, the cookie |
| [HOT_CODE_UPGRADES](beam/HOT_CODE_UPGRADES.md) | appup/relup, release_handler, current vs permanent |

### Elixir — `elixir/`

Language-specific knowledge.

| Note | Covers |
|------|--------|
| [README](elixir/README.md) | The language in one page |
| [MACROS](elixir/MACROS.md) | Quote/unquote, `use`, hygiene, macros are for libraries |
| [PROTOCOLS](elixir/PROTOCOLS.md) | Dispatch by first argument's type, `@derive`, consolidation |
| [STRUCTS](elixir/STRUCTS.md) | Fixed keys, enforced keys, structs vs maps |
| [MIX](elixir/MIX.md) | Project layout, tasks, releases |

### Erlang — `erlang/`

Language-specific knowledge.

| Note | Covers |
|------|--------|
| [README](erlang/README.md) | The language in one page |
| [MODULES_AND_RECORDS](erlang/MODULES_AND_RECORDS.md) | Exports, attributes, records as tagged tuples |
| [PARSE_TRANSFORMS](erlang/PARSE_TRANSFORMS.md) | Compile-time AST rewrites |

### Event Sourcing & CQRS — `event-sourcing/`

The write model, the read models, and everything between.

| Note | Covers |
|------|--------|
| [README](event-sourcing/README.md) | The pattern landscape at a glance |
| [EVENT_SOURCING](event-sourcing/EVENT_SOURCING.md) | State derived from events; replay; reinterpretation; trade-offs |
| [EVENTS](event-sourcing/EVENTS.md) | Past tense, atomicity, immutability |
| [COMMANDS_AND_QUERIES](event-sourcing/COMMANDS_AND_QUERIES.md) | Commands return status only; queries never mutate |
| [CQRS](event-sourcing/CQRS.md) | The one-line split; what it is not |
| [EVENT_LOGS](event-sourcing/EVENT_LOGS.md) | Appending, segmented, distributed; streams; log vs sourcing |
| [PROJECTIONS](event-sourcing/PROJECTIONS.md) | Simple, updating, inserting, batched; replay vs live |
| [CHECKPOINTS](event-sourcing/CHECKPOINTS.md) | Memory, file, database, stream, shared, consensus |
| [SAGAS_AND_PROCESS_MANAGERS](event-sourcing/SAGAS_AND_PROCESS_MANAGERS.md) | The distinction; versioning long processes |
| [IDS_AND_CORRELATION](event-sourcing/IDS_AND_CORRELATION.md) | Message, causation, correlation, conversation ids |
| [INTEGRATION_EVENTS](event-sourcing/INTEGRATION_EVENTS.md) | Denormalized contracts; aggregators; direct integration |
| [STREAM_FORK_AND_JOIN](event-sourcing/STREAM_FORK_AND_JOIN.md) | Reindexing with link events; join ordering |

### Testing — `testing/`

How to hold all of the above to a standard.

| Note | Covers |
|------|--------|
| [README](testing/README.md) | The three highest-value test shapes |
| [TESTING_EVENT_SOURCING](testing/TESTING_EVENT_SOURCING.md) | Given/when/then, replay tests, tests as documentation |
| [PROPERTY_BASED_TESTING](testing/PROPERTY_BASED_TESTING.md) | Invariants, generators, shrinking, stateful models |

---

## Reading path — agent

1. [`GLOSSARY.md`](GLOSSARY.md) for the vocabulary.
2. The domain README closest to the question.
3. The specific note, if the README points at one.

## Reading path — human

1. [`README.md`](README.md) → this index → the domain that interests you.
2. Notes are short and standalone; there is no required order.
