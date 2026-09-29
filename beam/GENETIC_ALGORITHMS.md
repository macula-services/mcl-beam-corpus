---
title: "BEAM: Genetic Algorithms"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Genetic Algorithms

*Evolution as a search strategy: populations of candidate solutions, selected and recombined. The BEAM's processes make the population parallel for free.*

---

## The loop

A genetic algorithm searches a solution space the way evolution does:

```
initialize population
repeat:
    evaluate fitness of each chromosome
    select parents (fit individuals reproduce)
    crossover + mutate → children
    replace the weakest with the children
until: good enough, or generations exhausted
```

Each **chromosome** encodes one candidate solution; the **fitness
function** is the problem distilled to a number. Everything else is
mechanics with many dialects.

---

## The vocabulary

| Term | Meaning |
|------|---------|
| Chromosome | One candidate solution, encoded (a list of genes) |
| Gene | One decision inside the chromosome — a city in a route, a parameter |
| Fitness | The score: how good this solution is |
| Selection | Choosing parents, biased toward fitness (tournament, roulette) |
| Crossover | Two parents exchange genes → children |
| Mutation | Random gene change — the source of novelty |
| Termination | Generations exhausted, or fitness plateaued |

---

## Why the BEAM is a natural fit

- **The population is embarrassingly parallel.** Fitness evaluation
  of N chromosomes is N independent tasks — `Task.async_stream`
  over the population, no shared state, all cores busy.
- **The state machine is a GenServer.** The GA loop — evaluate,
  select, breed — is a `handle_cast`/`handle_info` cycle holding the
  population as state; the book's structure follows exactly this.
- **Immutable data fits.** Chromosomes are immutable terms;
  crossover produces new terms, and the old generation simply falls
  out of scope.

## Rules of thumb

- **The fitness function is the whole problem.** Encode it carefully;
  everything else is interchangeable mechanics.
- **Diversity is the engine.** Mutation rate too high = random
  search; too low = premature convergence. Track diversity, not just
  best fitness.
- **Premature convergence is the classic failure.** The population
  collapses onto a local optimum; selection pressure and mutation
  balance against it.
- **Bench on the target problem.** No GA parameter set transfers
  between problems; measure generations-to-good-enough, then tune.

## Why it matters

Optimisation problems the mesh might face — scheduling, routing,
parameter search — have a GAs-shaped answer when the space is too
rough for gradients and too large for enumeration. The BEAM version
is the natural one: the population parallelised across schedulers,
the loop as a supervised process, the whole search restartable by
crash.
