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

```
my_app/
├── mix.exs          # project definition: version, deps, application config
├── lib/             # application source
├── test/            # ExUnit tests
├── config/          # config.exs + environment configs
└── priv/            # runtime assets, NIF binaries
```

`mix.exs` defines the project in `project/0`, the application in
`application/0`, and the dependencies in `deps/0`:

```elixir
def project do
  [app: :my_app, version: "0.1.0", deps: deps()]
end

def application do
  [mod: {MyApp, []}, extra_applications: [:logger]]
end

defp deps do
  [{:httpoison, "~> 2.0"}]
end
```

---

## The tasks worth knowing

| Task | Does |
|------|------|
| `mix new` | Scaffold a project (add `--sup` for the supervision tree) |
| `mix deps.get` | Fetch and lock dependencies (`mix.lock`) |
| `mix compile` | Compile with parallelisation and incremental recompiles |
| `mix test` | Run ExUnit |
| `mix format` | The opinionated formatter — run it, never argue with it |
| `mix xref` | Cross-reference checks: unused deps, undefined calls |
| `mix release` | Assemble a self-contained OTP release |
| `mix escript.build` | Build a single-file CLI executable |

Custom tasks are modules named `Mix.Tasks.X` with a `run/1` — they are
how repos script themselves (`mix ecto.migrate`).

---

## Releases

`mix release` bundles the ERTS, all applications, and the runtime
config into a directory that runs with **no source and no installed
runtime**:

```
_build/prod/rel/my_app/
└── bin/my_app start | stop | restart | remote
```

`bin/my_app remote` attaches an IEx shell to the running node — the
standard operator's door into production. Add `:runtime_tools` to
`extra_applications` to make `:observer` work against the release.

---

## Rules of thumb

- **Lock the deps.** `mix.lock` is committed for applications; it is
  what makes a build reproducible.
- **Config belongs in `config/`, not in code.** `System.fetch_env!`
  for secrets — never bake them into the release.
- **`mix format` in CI.** The formatter ends style debates; the CI
  check keeps the tree formatted.
- **Custom tasks stay thin.** A task parses args and calls a module; it
  is glue, not logic — the logic must be testable without the CLI.

## Why it matters

mix is how an Elixir project goes from `mix new` to a release that
runs on a fleet node. Every convention above — lockfile, config
directory, formatter, release — exists so that "how does this build?"
has one answer per project, not one per developer.
