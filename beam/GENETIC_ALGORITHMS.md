---
title: "BEAM: Genetic Algorithms"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Genetic Algorithms

*Search by simulated evolution: keep a population of candidate answers, favour the better ones, recombine and perturb them. Fitness evaluation parallelises naturally across BEAM processes.*

---

## What it is

A genetic algorithm (GA) is a stochastic optimiser for problems where
you can score a candidate but cannot easily compute the best one:
schedules, routes, parameter sets, layouts. It needs no gradient and
tolerates rugged, discontinuous search spaces. The price is that it
gives no optimality guarantee and needs many evaluations.

| Term | Meaning |
|------|---------|
| Genotype / chromosome | the encoded candidate, e.g. a list of integers or a permutation |
| Fitness | a number saying how good a candidate is; the whole problem lives here |
| Selection | choosing parents with a bias toward fitness (tournament, rank, roulette) |
| Crossover | building children from parts of two parents |
| Mutation | small random changes that keep variety in the population |
| Elitism | copying the best few unchanged into the next generation |

## The loop

```elixir
def evolve(population, gen, opts) do
  scored = Task.async_stream(population, &{opts.fitness.(&1), &1}, ordered: false)
           |> Enum.map(fn {:ok, pair} -> pair end)

  if done?(scored, gen, opts) do
    best(scored)
  else
    scored |> next_generation(opts) |> evolve(gen + 1, opts)
  end
end
```

`next_generation/2` applies selection, crossover and mutation; `done?/3`
stops on a fitness target, a generation cap, or a plateau.

---

## Why the BEAM suits it

- **Evaluation is independent per candidate,** so `Task.async_stream`
  spreads it over every scheduler with bounded concurrency
  ([TASKS_AND_AGENTS](TASKS_AND_AGENTS.md), [SCHEDULER](SCHEDULER.md)).
- **Immutable data makes generations cheap to reason about:** a new
  population is a new value, the old one is garbage.
- **Long runs can be supervised processes** that checkpoint the
  population, so a crash resumes instead of starting over
  ([GENSERVER](GENSERVER.md), [SUPERVISION_TREES](SUPERVISION_TREES.md)).
- **Island models map onto nodes:** separate populations evolving on
  different nodes and occasionally exchanging migrants.

## Pitfalls

- **Premature convergence.** The population collapses onto one local
  optimum. Watch diversity, not only best fitness; adjust selection
  pressure and mutation rate.
- **Encoding matters as much as operators.** A representation where
  small changes produce valid, similar candidates makes crossover and
  mutation useful; otherwise the GA is random search.
- **Fitness cost dominates.** If one evaluation is expensive, cache
  results and prefer fewer, larger generations.
- **Parameters do not transfer.** Population size and rates must be tuned
  per problem, with fixed random seeds so runs are reproducible.
- **CPU-heavy fitness in NIFs** must run on dirty schedulers.

## Sources

- *Genetic Algorithms in Elixir: Solve Problems Using Evolution*, 1st edition, Sean Moriarity, Pragmatic Bookshelf, 2021. <https://pragprog.com/titles/smgaelixir/genetic-algorithms-in-elixir/>
- Elixir `Task` documentation (`async_stream/3`). <https://elixir.hexdocs.pm/Task.html>
