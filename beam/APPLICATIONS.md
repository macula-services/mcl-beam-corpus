---
title: "BEAM: OTP Applications"
layer: guide
audience: [agent, human]
stage: stable
---

# BEAM: OTP Applications

*An application is the unit of composition on the BEAM: a supervision tree plus metadata. It starts together, stops together, and declares what it needs.*

---

## What an application is

An OTP **application** is a component of an OTP system: a supervision
tree (the runtime part) plus an application resource file (the metadata).
The runtime is what runs; the metadata says how it starts, stops, and
what it depends on.

The application is the unit the release system assembles: a release
bundles applications with the runtime so a node starts with everything
it needs and nothing it does not.

---

## The resource file

```erlang
{application, my_app,
 [{description, "..."},
  {vsn, "1.0.0"},
  {mod, {my_app, []}},              % application callback module
  {registered, [my_app_sup]},
  {applications, [kernel, stdlib]}, % runtime deps, started first
  {env, []},
  {modules, []}]}.
```

In Elixir, `mix` generates this from `application/0` in `mix.exs`:

```elixir
def application do
  [
    mod: {MyApp, []},
    extra_applications: [:logger]
  ]
end
```

---

## Start types

| Type | Meaning |
|------|---------|
| `:permanent` | Crash terminates the whole node — the default for normal apps |
| `:temporary` | Crash does not affect the node — for tools and one-shots |
| `:transient` | Normal exit is fine; abnormal crash takes the node down |

---

## The callback

```elixir
defmodule MyApp do
  use Application

  def start(_type, _args) do
    children = [MyApp.Supervisor]
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```

`start/2` must return `{:ok, pid}` of the top supervisor. The
application is "started" when its top supervisor is alive — everything
below it is the supervisor's business, not the application's.

---

## Dependencies and shutdown order

The `applications` list is the runtime dependency declaration: kernel and
stdlib start first, then the listed apps, then this one. Shutdown is the
reverse — children of the tree stop first, then the application, then its
dependencies. The whole system's lifecycle is a tree fold, which is why
"stop the node cleanly" is not something each process implements.

## Rules of thumb

- **One top supervisor per application.** `mod` points at the application
  callback; the callback starts exactly one supervisor.
- **Applications declare deps, not order-of-code.** Keep `applications`
  truthful: a missing dependency is a boot-time error, a false one is a
  lie future readers will build on.
- **Library vs runtime applications.** An application with no `mod` is a
  library — it has modules, no process tree, and starts trivially.
  Know which yours is; a library should never spawn a tree.
