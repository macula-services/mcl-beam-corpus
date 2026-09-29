---
title: "Architecture: Pattern Survey"
layer: guide
audience: [agent, human]
stage: stable
---

# Architecture: Pattern Survey

*Layered, microkernel, microservices, space-based. What each is, what it costs, and which one your system is asking for.*

---

## The four patterns (event-driven is its own note)

| Pattern | Shape | Sweet spot |
|---------|-------|------------|
| **Layered** | Presentation → business → persistence → database | Small, simple applications; rapid development |
| **Microkernel** | Core system + plug-in modules | Product-based apps: scheduled tasks, plugin ecosystems |
| **Microservices** | Independently deployed services, one bounded purpose | Highly decoupled domains, per-service scaling |
| **Space-based** | In-memory data grid, no central database | High-volume, variable-load, database-bottlenecked systems |

See [EVENT_DRIVEN_ARCHITECTURE](EVENT_DRIVEN_ARCHITECTURE.md) for the fifth pattern of the survey.

---

## The analysis axes

Every pattern is a trade on the same axes:

| Axis | Question |
|------|----------|
| Agility | How cheap is a change? |
| Deployability | What must ship together? |
| Testability | What can be tested alone? |
| Scalability | Where does load hit first? |
| Performance | Where does latency live? |
| Cost of getting it wrong | How painful is the wrong choice? |

**Layered** wins on simplicity and loses on agility: every layer knows
its neighbour, and the database couples everything. The **sinkhole
anti-pattern** — requests passing through layers that do nothing —
is the sign a layer exists for show.

**Microkernel** shines when the product *is* extensibility: the core
stays stable, plugins evolve. The cost: the core's contract is the
whole product, and getting it wrong is expensive.

**Microservices** buy independent evolution and per-service scaling,
and bill you in deployment, network, and testing complexity. A
monolith is not the failure mode; a badly cut microservice boundary
is.

**Space-based** removes the database as bottleneck with in-memory
grids and replication — at the cost of a fundamentally different data
model and real complexity.

---

## Choosing

1. **Start layered** — it is the cheapest thing that works for small
   systems, and its failure mode (sinkholes, coupled layers) is visible
   early.
2. **Cut to microservices by bounded context**, not by layer — see
   [DOMAIN_MODELING](../event-sourcing/DOMAIN_MODELING.md). A service
   per context scales; a service per layer doubles your latency.
3. **Reach for microkernel** when extensions by third parties are the
   product.
4. **Reach for space-based** only when measurements show the database
   is the bottleneck under your load — never on anticipation.
5. Event-driven topology is orthogonal to all four: any of them can be
   event-driven on the inside or between services.
