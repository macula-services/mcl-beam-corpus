---
title: "BEAM: Supervision Trees"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: Supervision Trees

*The failure model: supervisors start, watch, and restart workers. Supervisors never do domain work.*

---

## The model

A **supervisor** is a process whose only job is managing child processes:

1. Start children in a defined order.
2. Watch them (monitors, not links — a supervisor can restart without dying itself).
3. Restart them according to a **strategy** when they crash.
4. Shut them down in reverse order when the supervisor stops.

A **worker** is any process that does domain work: a `gen_server`, a
`gen_statem`, a `Task`. The tree is supervisors all the way down, workers
at the leaves.

**Let it crash.** Code handles the errors it expects and lets everything
else crash. The supervisor restarts from a known state. This is why BEAM
systems recover: state lives in processes that get rebuilt, not in
hand-rolled cleanup paths.

---

## Restart strategies

| Strategy | On one child crashing |
|----------|-----------------------|
| `one_for_one` | Restart only that child. The default for independent workers. |
| `one_for_all` | Restart all children. Use when children must start together (one depends on another's state). |
| `rest_for_one` | Restart the crashed child and everything started after it. Use when children are ordered, later depends on earlier. |

OTP 26 deprecated `:simple_one_for_one`; a supervisor that starts children
on demand is now a **DynamicSupervisor** (`one_for_one`, no static child
list, `start_child/2` at runtime).

---

## Child spec

Every child is described by a spec:

```elixir
%{
  id: MyWorker,              # unique within this supervisor (term, conventionally the module)
  start: {MyWorker, :start_link, [arg]},
  restart: :permanent,       # :permanent (always restart) | :transient (restart on abnormal exit) | :temporary (never)
  shutdown: 5_000,           # ms to wait in shutdown before :brutal_kill; :infinity | :brutal_kill
  type: :worker              # :worker | :supervisor
}
```

**Restart intensity.** The supervisor gives up on a child that crashes too
often: `max_restarts` restarts within `max_seconds` (defaults 3 in 5 s)
terminates the supervisor itself, and the failure propagates upward. A
crash loop must kill its supervisor — that is the design, not a bug.

---

## The shape of a tree

```elixir
Application
└── App.Supervisor                       # strategy: one_for_one
    ├── Registry (supervisor)            # names processes
    ├── EventStore.WorkerPool            # worker: gen_server
    ├── Repo.Supervisor                  # strategy: one_for_all
    │   ├── Repo.Connection              # worker
    │   └── Repo.Pool                    # worker — restarts with its sibling
    └── DynamicSupervisor                # workers on demand: one per user session
```

**Rules of thumb**

- Supervisors **do no work**: no domain logic in `init/1`, no business state.
- Children start **in order** and stop in reverse — encode dependencies in the order.
- One supervisor strategy per level; nest for mixed needs.
- `:permanent` for everything you cannot afford to lose; `:temporary` for one-shot tasks.
- A child that needs to survive the supervisor's restart belongs one level up or in a separate supervisor.

---

## Why this matters

A supervision tree is a **restart policy as data**, not code. Recovery is
uniform: every failure in the system is handled by the same mechanism,
which is why crash handling cannot be forgotten per call site. This is
the property fleets depend on: a node recovers from any single fault by
restarting the smallest subtree that needs it.
