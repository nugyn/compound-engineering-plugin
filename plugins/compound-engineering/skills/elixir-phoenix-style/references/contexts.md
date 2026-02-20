# Phoenix Contexts

## The Context Pattern

Contexts are modules that expose a public API for a domain. They hide schema details from the rest of the app.

```
MyApp.Accounts      # Public: create_user, get_user, authenticate
  └── User          # Private schema
  └── Token         # Private schema

MyApp.Trails        # Public: create_trail, list_trails, pause_trail
  └── Trail         # Private schema

MyApp.Captures      # Public: capture_page, list_pages, delete_page
  └── Page          # Private schema
```

**Rules:**
- Only the owning context imports its schemas
- Other contexts call the public API, never touch schemas directly
- Context functions return `{:ok, struct}` / `{:error, changeset}` consistently

## Structure

```elixir
defmodule MyApp.Trails do
  @moduledoc "Public API for the Trails domain."

  import Ecto.Query, warn: false

  alias MyApp.Repo
  alias MyApp.Trails.Trail

  # --- Queries ---

  def list_trails(user_id) do
    Trail
    |> where([t], t.user_id == ^user_id)
    |> order_by([t], desc: t.inserted_at)
    |> Repo.all()
  end

  def get_trail(id, user_id), do: Repo.get_by(Trail, id: id, user_id: user_id)

  def get_active_trail(user_id) do
    Repo.get_by(Trail, user_id: user_id, status: "active")
  end

  # --- Mutations ---

  def create_trail(attrs \\ %{}) do
    # Multi-step: auto-pause active trail first
    ...
  end

  def pause_trail(%Trail{} = trail) do
    trail
    |> Trail.status_changeset("paused")
    |> Repo.update()
  end

  def complete_trail(%Trail{} = trail) do
    trail
    |> Trail.status_changeset("completed")
    |> Repo.update()
  end
end
```

## Keep Business Logic in Changesets and Contexts

```elixir
# ✅ Business logic in changeset (pure, testable)
def status_changeset(trail, new_status) do
  trail
  |> change()
  |> put_change(:status, new_status)
  |> validate_status_transition()
  |> maybe_set_ended_at(new_status)
end

# ✅ Orchestration in context
def complete_trail(%Trail{} = trail) do
  trail
  |> Trail.status_changeset("completed")
  |> Repo.update()
end

# ❌ Business logic leaked into LiveView or controller
def handle_event("complete", %{"id" => id}, socket) do
  trail = Trails.get_trail!(id)
  trail
  |> Ecto.Changeset.change(%{status: "completed", ended_at: DateTime.utc_now()})
  |> Repo.update()
  ...
end
```

## Cross-Context Communication

When contexts need to interact, call public APIs — not schemas:

```elixir
# ✅ Captures context calls Trails public API
defmodule MyApp.Captures do
  alias MyApp.Trails  # context module, not schema

  def capture_page(attrs) do
    trail_id = attrs["trail_id"]
    # Verify via public API
    case Trails.get_trail(trail_id, attrs["user_id"]) do
      nil -> {:error, :trail_not_found}
      _trail -> insert_page(attrs)
    end
  end
end

# ❌ Never reach into another context's schema
defmodule MyApp.Captures do
  alias MyApp.Trails.Trail   # WRONG
  def get_trail_name(id), do: Repo.get!(Trail, id).name
end
```

## PubSub in Contexts

Broadcast domain events from the context, not from LiveView:

```elixir
defmodule MyApp.Captures do
  alias MyApp.PubSub

  def capture_page(attrs) do
    with {:ok, page} <- insert_page(attrs) do
      broadcast_page_captured(page)
      {:ok, page}
    end
  end

  defp broadcast_page_captured(page) do
    Phoenix.PubSub.broadcast(
      PubSub,
      "trail:#{page.trail_id}",
      {:page_captured, page}
    )
  end
end
```
