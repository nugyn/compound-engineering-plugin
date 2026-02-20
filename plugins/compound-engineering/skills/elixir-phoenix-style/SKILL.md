---
name: elixir-phoenix-style
description: This skill should be used when writing Elixir and Phoenix applications following idiomatic OTP, Ecto, and LiveView patterns. It applies when writing Elixir code, Phoenix controllers, LiveView modules, Ecto schemas or changesets, or any `.ex`/`.exs` file. Triggers on Elixir/Phoenix code generation, refactoring requests, code review, or when the user mentions Phoenix, LiveView, Ecto, OTP, GenServer, or Elixir conventions.
---

<objective>
Apply idiomatic Elixir, Phoenix, and OTP conventions. This skill provides domain expertise for Phoenix contexts, Ecto changesets, LiveView state management, OTP supervision, and ExUnit testing — grounded in real production patterns.
</objective>

<essential_principles>
## Core Philosophy

"Make it work, make it right, make it fast — in that order. The database is your source of truth, not your processes."

**Elixir way:**
- Contexts own their schemas — no cross-context schema access
- Changesets are the validation layer — pure, composable, testable
- Database as source of truth — no GenServers for domain entities
- Let It Crash — supervision trees, not defensive rescue blocks
- Transactions for atomicity — `Repo.transaction/1` or `Ecto.Multi`
- Pattern match on results — `{:ok, _}` / `{:error, _}` everywhere

**What to avoid:**
- GenServers for domain entities (Trail, User, Order, etc.) — use the DB
- Floats for money — use `integer` (cents) or `Decimal`
- `:utc_datetime` for ordered timestamps — always use `:utc_datetime_usec`
- Atoms as JSONB keys after DB round-trip — reload after insert or normalize keys
- `Repo.get!/1` without authorization — use scoped queries
- Inline business logic in controllers or LiveView `handle_event` — delegate to contexts
</essential_principles>

<intake>
What are you working on?

1. **Ecto** — Schemas, changesets, queries, migrations, transactions
2. **Phoenix Contexts** — Domain boundaries, service logic, Repo usage
3. **Controllers & API** — REST JSON API, fallback controller, auth pipelines
4. **LiveView** — Assigns, events, forms, PubSub, components
5. **OTP** — GenServers (infrastructure), supervisors, PubSub, Oban workers
6. **Testing** — ExUnit, DataCase, ConnCase, factories
7. **Code Review** — Review code against idiomatic Elixir/Phoenix patterns
8. **General Guidance** — Philosophy, naming, project structure

**Specify a number or describe your task.**
</intake>

<routing>

| Response | Reference to Read |
|----------|-------------------|
| 1, ecto, schema, changeset, migration, query, repo | [ecto.md](./references/ecto.md) |
| 2, context, domain, service, boundary | [contexts.md](./references/contexts.md) |
| 3, controller, api, json, auth, pipeline, route | [api.md](./references/api.md) |
| 4, liveview, live, assigns, handle_event, form, pubsub, component | [liveview.md](./references/liveview.md) |
| 5, otp, genserver, supervisor, oban, worker, process | [otp.md](./references/otp.md) |
| 6, test, testing, exunit, datacase, conncase, factory | [testing.md](./references/testing.md) |
| 7, review | Read all references, then review code |
| 8, general task | Read relevant references based on context |

**After reading relevant references, apply patterns to the user's code.**
</routing>

<quick_reference>
## Naming Conventions

**Contexts:** Plural nouns — `Accounts`, `Trails`, `Captures`, `Orders`

**Schemas:** Singular nouns — `User`, `Trail`, `Page`, `Order`

**Context functions:** `verb_noun` — `create_trail/1`, `get_trail/2`, `list_trails/1`, `pause_trail/1`

**Changesets:** Named for their purpose — `create_changeset/1`, `status_changeset/2`, `password_changeset/2`

**LiveView modules:** `{Feature}Live` — `TrailShowLive`, `TrailListLive`, `UserSettingsLive`

**Components:** `{noun}_{variant}` functions — `trail_card/1`, `status_badge/1`, `flash/1`

## Project Structure

```
lib/
├── my_app/                   # Business logic
│   ├── accounts/             # Auth context schemas
│   │   └── user.ex
│   ├── accounts.ex           # Accounts context public API
│   ├── trails/               # Trail domain schemas
│   │   └── trail.ex
│   ├── trails.ex             # Trails context public API
│   ├── application.ex
│   └── repo.ex
└── my_app_web/               # Web layer
    ├── components/
    │   ├── core_components.ex
    │   └── layouts.ex
    ├── controllers/
    │   └── api/              # JSON API controllers
    ├── live/                 # LiveView pages
    ├── router.ex
    └── endpoint.ex
```

## The Context Public API Pattern

```elixir
# ✅ Context owns and exposes clean public functions
defmodule MyApp.Trails do
  alias MyApp.Repo
  alias MyApp.Trails.Trail

  def get_trail(id, user_id), do: Repo.get_by(Trail, id: id, user_id: user_id)
  def list_trails(user_id), do: Trail |> where(user_id: ^user_id) |> Repo.all()
  def create_trail(attrs), do: Trail.create_changeset(attrs) |> Repo.insert()
end

# ❌ Never reach into another context's schema
defmodule MyApp.Captures do
  alias MyApp.Trails.Trail   # WRONG - cross-context schema access
end
```

## Error Handling

```elixir
# ✅ Pattern match, don't rescue
def handle_event("save", params, socket) do
  case Trails.create_trail(params) do
    {:ok, trail}    -> {:noreply, push_navigate(socket, to: ~p"/trails/#{trail}")}
    {:error, cs}    -> {:noreply, assign(socket, form: to_form(cs))}
  end
end

# ❌ Don't rescue from expected errors
def create(conn, params) do
  try do
    trail = Trails.create_trail!(params)
    json(conn, trail)
  rescue
    e -> json(conn, %{error: e.message})
  end
end
```
</quick_reference>
