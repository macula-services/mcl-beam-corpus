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

An OTP **application** is a named component with a version, a set of
modules, an optional supervision tree and a resource file describing
it. The runtime part is what runs; the metadata says how it starts,
what configuration it has and which other applications must be running
first.

Applications are what the release system assembles: a release bundles a
chosen set of application versions with the runtime, so a node boots
with exactly what it needs.

---

## The resource file

```erlang
{application, billing,
 [{description, "Invoice issuing"},
  {vsn, "1.4.0"},
  {modules, [billing_app, billing_sup, billing_ledger]},
  {registered, [billing_sup]},
  {applications, [kernel, stdlib, crypto]},  % must be started before this one
  {mod, {billing_app, []}},                  % callback module and start argument
  {env, [{currency, eur}]}]}.
```

In Elixir, Mix generates this `.app` file from `application/0` in
`mix.exs`; runtime dependencies are inferred from `deps` and extended
with `extra_applications`:

```elixir
def application do
  [mod: {Billing.Application, []}, extra_applications: [:logger, :crypto]]
end
```

---

## The callback

```elixir
defmodule Billing.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [Billing.Ledger, {Task.Supervisor, name: Billing.Tasks}]
    Supervisor.start_link(children, strategy: :one_for_one, name: Billing.Supervisor)
  end
end
```

`start/2` returns `{:ok, pid}` (or `{:ok, pid, state}`) where `pid` is
the top supervisor. The application is running while that supervisor
is alive; everything below it is the tree's business
([SUPERVISION_TREES](SUPERVISION_TREES.md)).

---

## Start types

The type decides what happens to the node when the application's top
supervisor terminates:

| Type | If the application terminates |
|------|-------------------------------|
| `temporary` | Reported, nothing else stops. The default for `application:start/1` and `Application.start/1`. |
| `transient` | Normal exit is reported only; any other reason terminates all applications and the node. |
| `permanent` | All other applications and the node terminate. Releases normally start their applications this way. |

---

## Dependencies and shutdown

The `applications` list is checked at start: listed applications must
already be running (`application:ensure_all_started/1` starts them in
dependency order). Stopping an application terminates its top
supervisor, which stops children in reverse start order. When the node
shuts down, applications stop in the reverse of the order they started.
Clean shutdown is therefore a property of the tree, not something each
process implements.

## Rules of thumb

- **One top supervisor per application,** started by the callback module.
- **Keep `applications` truthful.** A missing dependency fails at boot or
  worse, at first use; an unnecessary one misleads the next reader.
- **Library applications have no `mod`.** They provide modules and no
  process tree. A library should not start processes on load.
- Configuration read at runtime goes through the application
  environment (`Application.fetch_env!/2`), set from config files or
  release runtime configuration ([MIX](../elixir/MIX.md)).

## Sources

- Erlang/OTP Design Principles, Applications. <https://www.erlang.org/doc/system/applications.html>
- Kernel `app` file reference. <https://www.erlang.org/doc/apps/kernel/app.html>
- Kernel `application` reference (start types, callback returns). <https://www.erlang.org/doc/apps/kernel/application.html>
- Elixir `Application` documentation. <https://elixir.hexdocs.pm/Application.html>
