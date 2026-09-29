---
title: "Elixir: mix"
layer: guide
audience: [agent, human]
stage: stable
---

# Elixir: mix

*The build tool: tasks, dependencies, tests, releases. One tool from scaffold to production bundle.*

---

## The project layout

```text
my_app/
├── mix.exs          # project definition: version, deps, application config
├── lib/             # application source
├── test/            # ExUnit tests
├── config/          # config.exs, per-environment files, runtime.exs
└── priv/            # runtime assets, migrations, NIF binaries
```

`mix.exs` defines the project in `project/0`, the application in
`application/0` (from which Mix writes the `.app` resource file, see
[APPLICATIONS](../beam/APPLICATIONS.md)), and dependencies in `deps/0`:

```elixir
def project do
  [app: :my_app, version: "0.1.0", elixir: "~> 1.17", deps: deps()]
end

def application do
  [mod: {MyApp.Application, []}, extra_applications: [:logger]]
end

defp deps do
  [{:jason, "~> 1.4"}]
end
```

---

## The tasks worth knowing

| Task | Does |
|------|------|
| `mix new` | Scaffold a project (`--sup` adds an application callback and supervisor) |
| `mix deps.get` | Fetch dependencies and record resolved versions in `mix.lock` |
| `mix compile` | Incremental compilation; also consolidates protocols ([PROTOCOLS](PROTOCOLS.md)) |
| `mix test` | Run ExUnit |
| `mix format` | Apply the standard formatter; `--check-formatted` for CI |
| `mix xref` | Inspect the dependency graph between modules (`mix xref graph`, `callers`) |
| `mix deps.unlock --check-unused` | Fail if the lockfile holds dependencies no longer used |
| `mix release` | Assemble a self-contained OTP release |
| `mix escript.build` | Build a single-file executable (needs Erlang installed to run) |

Custom tasks are modules named `Mix.Tasks.Something` that `use Mix.Task`
and implement `run/1`; that is how libraries add commands such as
`mix ecto.migrate`.

---

## Releases

`mix release` bundles the Erlang runtime (ERTS, included by default),
all applications and their compiled code into a directory that runs
without Elixir or Erlang installed on the target:

```text
_build/prod/rel/my_app/bin/my_app start | start_iex | daemon | remote | rpc | eval | stop | restart | pid | version
```

`bin/my_app remote` opens an IEx shell connected to the running node,
the usual operator entry point. `config/runtime.exs` is evaluated when
the release boots, which is where environment variables and secrets
are read (`System.fetch_env!/1`); build-time config files are baked in.
Releases built with Mix do not support hot code upgrades out of the box
([HOT_CODE_UPGRADES](../beam/HOT_CODE_UPGRADES.md)).

---

## Rules of thumb

- **Control versions through constraints.** The wider Elixir convention
  is to commit `mix.lock` for applications. Macula repositories do not
  commit BEAM lockfiles (`mix.lock`, `rebar.lock`): versions are
  controlled by the constraints in `deps/0`, and a lock is never edited
  by hand. The consequence is that a fresh build can resolve newer
  matching versions, so a runtime result proves only the build that
  produced it; check what a tagged build actually bundled.
- **Secrets at runtime, never at build time.** Read them in
  `config/runtime.exs`, not in `config/config.exs` or module attributes.
- **`mix format --check-formatted` in CI** ends style debates.
- **Custom tasks stay thin.** Parse arguments, call a module; the logic
  must be testable without the CLI.

## Sources

- Mix documentation. <https://mix.hexdocs.pm/Mix.html>
- Mix `mix release` documentation. <https://mix.hexdocs.pm/Mix.Tasks.Release.html>
- Mix `mix deps.get` documentation. <https://mix.hexdocs.pm/Mix.Tasks.Deps.Get.html>
- Elixir `Application` documentation (Mix-generated resource file). <https://elixir.hexdocs.pm/Application.html>
