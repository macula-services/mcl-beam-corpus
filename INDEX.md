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

### Elixir — `elixir/`

Language-specific knowledge.

| Note | Covers |
|------|--------|
| [README](elixir/README.md) | What belongs here, planned notes |

### Erlang — `erlang/`

Language-specific knowledge.

| Note | Covers |
|------|--------|
| [README](erlang/README.md) | What belongs here, planned notes |

### Event Sourcing & CQRS — `event-sourcing/`

The write model, the read models, and everything between.

| Note | Covers |
|------|--------|
| [README](event-sourcing/README.md) | The pattern landscape at a glance |
| [PROJECTIONS](event-sourcing/PROJECTIONS.md) | Deriving read models from events: simple, updating, batched |

### Testing — `testing/`

How to hold all of the above to a standard.

| Note | Covers |
|------|--------|
| [README](testing/README.md) | What belongs here, planned notes |

---

## Reading path — agent

1. [`GLOSSARY.md`](GLOSSARY.md) for the vocabulary.
2. The domain README closest to the question.
3. The specific note, if the README points at one.

## Reading path — human

1. [`README.md`](README.md) → this index → the domain that interests you.
2. Notes are short and standalone; there is no required order.

---

## Planned

- `beam/`: applications, ETS, distribution, hot code upgrades
- `event-sourcing/`: event logs, checkpoints, sagas & process managers, ids & correlation
- `testing/`: property-based testing, fault injection
