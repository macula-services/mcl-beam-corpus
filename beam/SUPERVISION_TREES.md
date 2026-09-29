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

1. Start children in the order of its child list.
2. Watch them: a supervisor traps exits and is **linked** to each child
   it starts, so a child's termination arrives as an exit signal it
   handles instead of dying.
3. Restart children according to a **strategy** when they terminate.
4. Shut children down in reverse start order when the supervisor stops.

A **worker** is any process that does domain work: a `gen_server`, a
`gen_statem`, a `Task`. The tree is supervisors at the branches and
workers at the leaves.

**Let it crash.** Code handles the errors it expects and lets everything
else crash. The supervisor restarts the process from a known initial
state. Recovery lives in one uniform mechanism instead of hand-rolled
cleanup paths.

---

## Restart strategies

| Strategy | When one child terminates |
|----------|---------------------------|
| `one_for_one` | Restart only that child. The default for independent workers. |
| `one_for_all` | Terminate and restart all children. For children that cannot work without each other. |
| `rest_for_one` | Restart that child and every child started after it. For ordered dependencies. |

Erlang's `supervisor` also has `simple_one_for_one` for many identical,
dynamically added children, and it is still supported there. Elixir
1.6 deprecated that strategy in Elixir's `Supervisor` and moved the use
case to **DynamicSupervisor**: no static child list, children started
with `DynamicSupervisor.start_child/2`, optional `:max_children`.

---

## Child spec

```elixir
%{
  id: MyWorker,                          # unique within this supervisor
  start: {MyWorker, :start_link, [arg]},
  restart: :permanent,                   # :permanent | :transient (restart on abnormal exit) | :temporary (never)
  shutdown: 5_000,                       # ms before a forced kill; or :brutal_kill | :infinity
  type: :worker                          # :worker | :supervisor
}
```

Defaults: `restart: :permanent`; `shutdown` is 5000 ms for workers and
`:infinity` for supervisors.

**Restart intensity.** If more than `max_restarts` restarts happen within
`max_seconds`, the supervisor terminates its children and then itself,
and the failure escalates to its own supervisor. Elixir's `Supervisor`
defaults to 3 restarts in 5 seconds; Erlang's `supervisor` defaults to
`intensity` 1 in a `period` of 5 seconds. A crash loop is meant to
escalate.

---

## The shape of a tree

```text
Application
└── App.Supervisor                       # one_for_one
    ├── Registry                         # names processes
    ├── EventStore.WorkerPool            # worker: gen_server
    ├── Repo.Supervisor                  # one_for_all
    │   ├── Repo.Connection              # worker
    │   └── Repo.Pool                    # worker, restarts with its sibling
    └── DynamicSupervisor                # workers on demand: one per user session
```

**Rules of thumb**

- Supervisors **do no work**: no domain logic in `init/1`, no business state.
- Children start **in order** and stop in reverse; encode dependencies in the order.
- One strategy per level; nest supervisors for mixed needs.
- `:permanent` for services that must always run; `:temporary` for one-shot tasks.
- A child that must survive its siblings' restarts belongs in a different subtree.

---

## Why this matters

A supervision tree is a **restart policy expressed as data**, not code.
Every failure is handled by the same mechanism, so crash handling cannot
be forgotten at a call site. Fleets depend on this property: a node
recovers from a single fault by restarting the smallest subtree that
needs it. See also [APPLICATIONS](APPLICATIONS.md), [GENSERVER](GENSERVER.md) and [TASKS_AND_AGENTS](TASKS_AND_AGENTS.md).

## Sources

- Erlang/OTP Design Principles, Supervisor Behaviour. <https://www.erlang.org/doc/system/sup_princ.html>
- Erlang/OTP `supervisor` reference. <https://www.erlang.org/doc/apps/stdlib/supervisor.html>
- Elixir `Supervisor` documentation. <https://elixir.hexdocs.pm/Supervisor.html>
- Elixir `DynamicSupervisor` documentation. <https://elixir.hexdocs.pm/DynamicSupervisor.html>
- Elixir v1.6 release announcement (DynamicSupervisor replaces `:simple_one_for_one`). <https://elixir-lang.org/blog/2018/01/17/elixir-v1-6-0-released/>
