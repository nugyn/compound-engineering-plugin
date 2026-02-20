# OTP Patterns

## Database as Source of Truth

The most important OTP rule for Phoenix apps:

```elixir
# ❌ NEVER store domain entity state in a GenServer
defmodule TrailServer do
  use GenServer
  # If this crashes, trail state is lost!
  # If two nodes run this, state is inconsistent!
  def init(trail_id), do: {:ok, %{trail_id: trail_id, status: "active"}}
end

# ✅ Always use the database
defmodule MyApp.Trails do
  def pause_trail(%Trail{} = trail) do
    trail |> Trail.status_changeset("paused") |> Repo.update()
    # State is durable, replicated, queryable — no GenServer needed
  end
end
```

## When GenServers ARE Appropriate

GenServers are for **infrastructure**, not domain entities:

```elixir
# ✅ Cache (transient data, can always rebuild from DB)
defmodule MyApp.Runtime.TrailCache do
  use GenServer
  # Acceptable to lose on crash — just reload from DB
end

# ✅ Rate limiter (counting, not domain state)
defmodule MyApp.Runtime.RateLimiter do
  use GenServer
  # Acceptable to lose counts on crash
end

# ✅ WebSocket connection tracking (ephemeral)
# Phoenix Presence handles this

# ✅ Scheduled job coordinator
# Oban handles this
```

## Application Supervision Tree

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,
      {Phoenix.PubSub, name: MyApp.PubSub},
      MyAppWeb.Endpoint,
      {Oban, Application.fetch_env!(:my_app, Oban)}
      # Add GenServers for infrastructure here, NOT domain entities
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

## PubSub

```elixir
# Broadcast from context (not from LiveView or controller)
defmodule MyApp.Captures do
  def capture_page(attrs) do
    with {:ok, page} <- insert_page(attrs) do
      Phoenix.PubSub.broadcast(MyApp.PubSub, "trail:#{page.trail_id}", {:page_captured, page})
      {:ok, page}
    end
  end
end

# Subscribe in LiveView mount
def mount(%{"id" => id}, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "trail:#{id}")
  end
  {:ok, socket}
end

# Handle in LiveView
def handle_info({:page_captured, page}, socket) do
  {:noreply, update(socket, :pages, fn pages -> [page | pages] end)}
end
```

## Oban Workers (Background Jobs)

Never block the request path with external calls — enqueue a job:

```elixir
# ✅ Define the worker
defmodule MyApp.Workers.NotificationWorker do
  use Oban.Worker, queue: :notifications, max_attempts: 3

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"user_id" => uid, "type" => type, "resource_id" => rid}}) do
    with {:ok, user} <- get_user(uid),
         {:ok, _}    <- send_notification(user, type, rid) do
      :ok
    end
  end
end

# ✅ Enqueue from context after DB write
def capture_page(attrs) do
  with {:ok, page} <- insert_page(attrs) do
    %{user_id: page.user_id, type: "page_captured", resource_id: page.id}
    |> MyApp.Workers.NotificationWorker.new()
    |> Oban.insert()

    {:ok, page}
  end
end
```

## Let It Crash

```elixir
# ✅ Pattern match, don't rescue expected errors
def pause_trail(trail) do
  trail
  |> Trail.status_changeset("paused")
  |> Repo.update()
end
# Caller handles {:ok, _} / {:error, _}

# ✅ rescue only truly unexpected errors (e.g. external APIs)
def fetch_metadata(url) do
  Req.get!(url)
rescue
  exception ->
    Logger.error("Unexpected HTTP error: #{Exception.message(exception)}")
    {:error, :fetch_failed}
end

# ❌ Don't rescue from Ecto or domain errors
def create_trail(attrs) do
  try do
    Repo.insert!(Trail.create_changeset(attrs))
  rescue
    Ecto.InvalidChangesetError -> {:error, :invalid}
  end
end
```
