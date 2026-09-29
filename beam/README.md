---
title: BEAM & OTP
layer: index
audience: [agent, human]
stage: stable
---

# BEAM & OTP

*The virtual machine, applications, supervisors, fault tolerance, distribution.*

Every service on the Macula mesh is an OTP release on the BEAM. These
notes are the reference for how that substrate works.

---

## Notes

| Note | Covers |
|------|--------|
| [SUPERVISION_TREES](SUPERVISION_TREES.md) | Failure model, supervisor strategies, child specs, DynamicSupervisor |
| [APPLICATIONS](APPLICATIONS.md) | The composition unit: resource files, start types, dependencies |
| [GENSERVER](GENSERVER.md) | The stateful process: callbacks, the boundary shape, call vs cast |
| [TASKS_AND_AGENTS](TASKS_AND_AGENTS.md) | One-shot async work, simple state, the hierarchy of choosing |
| [REGISTRY](REGISTRY.md) | Naming processes: unique/duplicate keys, the via pattern |
| [ETS](ETS.md) | In-memory tables: types, ownership, what it is and is not for |
| [SCHEDULER](SCHEDULER.md) | The m:n model, reductions, concurrency vs parallelism |
| [DISTRIBUTION](DISTRIBUTION.md) | Location transparency, nodes, EPMD, the cookie |
| [HOT_CODE_UPGRADES](HOT_CODE_UPGRADES.md) | appup/relup, release_handler, current vs permanent |
| [CONCURRENCY_MODELS](CONCURRENCY_MODELS.md) | The seven models; the actor model's place among them |
| [DESIGN_PATTERNS](DESIGN_PATTERNS.md) | The GoF 23 mapped onto BEAM built-ins |

## One-page summary

The BEAM runs millions of **processes**, each isolated: a crash in one
cannot corrupt another's memory. Processes talk only by **message
passing**. Failures are handled by **supervisors**, which restart crashed
workers from a known state — this is the **let it crash** model, and the
reason Erlang systems reach nines of uptime.

An **application** is the unit of composition: a supervision tree plus
start/stop callbacks. A **release** bundles applications with the runtime
so a node runs with no source code installed.
