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

## Planned

- Applications: what an OTP application is, start types, deps
- The scheduler: pre-emption, reductions, dirty schedulers, NIFs
- ETS: ownership, heirs, concurrency, when not to use it
- Distribution: nodes, cookies, global vs pg, when to avoid it
- Hot code upgrades: appups/relups, why they matter for fleets

## One-page summary

The BEAM runs millions of **processes**, each isolated: a crash in one
cannot corrupt another's memory. Processes talk only by **message
passing**. Failures are handled by **supervisors**, which restart crashed
workers from a known state — this is the **let it crash** model, and the
reason Erlang systems reach nines of uptime.

An **application** is the unit of composition: a supervision tree plus
start/stop callbacks. A **release** bundles applications with the runtime
so a node runs with no source code installed.
