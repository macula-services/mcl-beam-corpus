---
title: mcl-beam-corpus
layer: index
audience: [agent, human]
stage: stable
---

# mcl-beam-corpus

*Knowledge corpus for BEAM languages (Erlang, Elixir), OTP, Event Sourcing / CQRS, and the architecture above them. Markdown only — this repo is ingested by [mcl-rag](https://github.com/macula-services/mcl-rag), the mesh's shared memory.*

This repository holds reference knowledge an agent on the Macula mesh can
recall: how the BEAM works, how to structure OTP applications, how to build
event-sourced systems, and how to test all of it. It contains no runtime
code. It is one domain corpus of the mesh's federated retrieval.

> **mcl-rag ingestion notes.** The sync loop fast-forwards this repo and
> re-embeds changed `**/*.md` files every 120 s. A commit is a deploy:
> the new knowledge is searchable within two minutes. Every note is
> chunked as written, so keep headings tight and paragraphs short.

---

## Start here

| You are | Read |
|---------|------|
| Agent, first recall | [`INDEX.md`](INDEX.md) → domain map |
| Human, first contact | [`INDEX.md`](INDEX.md) → read top-down |
| Looking for a term | [`GLOSSARY.md`](GLOSSARY.md) |
| BEAM/OTP questions | [`beam/`](beam/README.md) |
| Elixir-specific questions | [`elixir/`](elixir/README.md) |
| Erlang-specific questions | [`erlang/`](erlang/README.md) |
| Event sourcing / CQRS questions | [`event-sourcing/`](event-sourcing/README.md) |
| Testing questions | [`testing/`](testing/README.md) |

---

## Layout

| Domain | Where | Purpose |
|--------|-------|---------|
| **BEAM & OTP** | `beam/` | The virtual machine, applications, supervisors, fault tolerance, distribution |
| **Elixir** | `elixir/` | The language: macros, protocols, structs, `mix` |
| **Erlang** | `erlang/` | The language: modules, records, parse transforms |
| **Event Sourcing & CQRS** | `event-sourcing/` | Events, projections, checkpoints, sagas, process managers |
| **Testing** | `testing/` | ExUnit, property-based testing, fault injection |
| **Architecture** | `architecture/` | Event-driven, layered, microservices — the system-level patterns |

---

## Conventions

- **Front-matter** on every note: `title`, `layer`, `audience`, `stage`.
- **One pattern per note.** Notes are lookup targets for RAG, not books.
  A note that covers two patterns is two notes.
- **Language-neutral where possible.** Event-sourcing notes describe the
  pattern; code snippets show Elixir only when it adds clarity.
- **`stage`** is `draft`, `stable`, or `superseded`. Superseded notes link
  to their replacement instead of being deleted, so old recalls still
  point somewhere.
- **License:** MIT. Deposit only knowledge you may license as MIT.
