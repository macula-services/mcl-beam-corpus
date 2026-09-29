---
title: "Architecture: Pattern Survey"
layer: guide
audience: [agent, human]
stage: stable
---

# Architecture: Pattern Survey

*Layered, microkernel, microservices, space-based: what each shape is, what it trades away, and what kind of system it suits.*

---

## The shapes at a glance

| Style | Structure | Suits |
|-------|-----------|-------|
| **Layered** | Horizontal tiers (presentation, business, persistence), each calling only the one below | Small or simple applications, teams new to a domain |
| **Microkernel** | A minimal core plus independently built plug-ins | Products whose value is extensibility: IDEs, rule engines, tools with add-ons |
| **Microservices** | Independently deployable services, each owning one capability and its data | Many teams, parts that must scale or change at different rates |
| **Space-based** | Processing units holding data in replicated memory; the database is written asynchronously, off the request path | Very high or spiky load where the database is the proven bottleneck |

Event-driven architecture is covered separately in
[EVENT_DRIVEN_ARCHITECTURE](EVENT_DRIVEN_ARCHITECTURE.md). It combines
with any of these shapes.

## Questions to ask of any style

| Quality | Question |
|---------|----------|
| Changeability | How much must change, and where, for one new feature? |
| Deployability | What has to be released together? |
| Testability | What can be tested in isolation? |
| Scalability | Which part saturates first, and can it scale alone? |
| Performance | How many hops and serialisations does a request cross? |
| Reversibility | How expensive is it to discover this was the wrong choice? |

## Each style, briefly

**Layered.** Easy to start and to understand. Its weakness is that a
feature cuts through every layer, so changes spread horizontally, and
layers tend to share one database. Watch for layers that only forward
calls without adding anything (often called the architecture sinkhole);
a few are harmless, many mean the layering is ceremony.

**Microkernel.** The core stays small and stable; features arrive as
plug-ins behind a contract. The contract is the product: changing it
breaks every plug-in, so it needs versioning discipline from the start.

**Microservices.** Independent deployment and scaling per capability, at
the price of network calls, distributed data, harder end-to-end testing,
and serious operational tooling. The most expensive mistake is a bad
service boundary, which turns every feature into a coordinated
multi-service release.

**Space-based.** Removes the central database from the hot path by
keeping working data in replicated in-memory grids and persisting
asynchronously. Scales elastically, but the consistency model, data
collisions between replicas, and testing at realistic load are all
hard.

## On the BEAM

Several of these ideas are built into OTP. An OTP application with its
supervision tree is already a modular unit; a release can be a
"monolith of applications" that is split into separately deployed nodes
later. ETS and replicated stores offer space-based-style in-memory data
without extra infrastructure, and distributed Erlang gives location
transparency between nodes. See
[APPLICATIONS](../beam/APPLICATIONS.md) and
[DISTRIBUTION](../beam/DISTRIBUTION.md).

## Choosing

1. **Start simple.** A modular monolith (layered or, better, sliced by
   capability) is cheap and its problems show early.
2. **Cut services along bounded contexts**, not along technical layers
   ([DOMAIN_MODELING](../event-sourcing/DOMAIN_MODELING.md)). A service
   per layer adds network hops to every request without adding
   independence.
3. **Choose microkernel** when third-party or optional extensions are
   central to the product.
4. **Choose space-based** only when measurement shows the database is
   the limit under real load.
5. **Treat event-driven integration as orthogonal**: any of the above can
   use events inside or between components.

## Sources

- Mark Richards, *Software Architecture Patterns*, 1st edition, O'Reilly Media, 2015 (https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/); 2nd edition, O'Reilly Media, 2022 (https://www.oreilly.com/library/view/software-architecture-patterns/9781098134280/). Author's page: https://developertoarchitect.com/publishedbooks.html
- James Lewis and Martin Fowler, "Microservices", martinfowler.com, 2014 (free). https://martinfowler.com/articles/microservices.html
- Microsoft, "Architecture styles", Azure Architecture Center (free). https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/
