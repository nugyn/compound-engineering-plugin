# Phoenix REST API Patterns

## Controller Structure

```elixir
defmodule MyAppWeb.Api.TrailController do
  use MyAppWeb, :controller

  alias MyApp.Trails

  action_fallback MyAppWeb.FallbackController

  def index(conn, _params) do
    user = conn.assigns.current_user
    trails = Trails.list_trails(user.id)
    render(conn, :index, trails: trails)
  end

  def show(conn, %{"id" => id}) do
    user = conn.assigns.current_user

    with %Trail{} = trail <- Trails.get_trail(id, user.id) do
      render(conn, :show, trail: trail)
    end
  end

  def create(conn, %{"trail" => params}) do
    user = conn.assigns.current_user

    with {:ok, trail} <- Trails.create_trail(Map.put(params, "user_id", user.id)) do
      conn
      |> put_status(:created)
      |> render(:show, trail: trail)
    end
  end

  def pause(conn, %{"id" => id}) do
    user = conn.assigns.current_user

    with %Trail{} = trail <- Trails.get_trail(id, user.id),
         {:ok, trail}     <- Trails.pause_trail(trail) do
      render(conn, :show, trail: trail)
    end
  end

  def delete(conn, %{"id" => id}) do
    user = conn.assigns.current_user

    with %Trail{} = trail <- Trails.get_trail(id, user.id),
         {:ok, _}         <- Trails.delete_trail(trail) do
      send_resp(conn, :no_content, "")
    end
  end
end
```

## Fallback Controller

```elixir
defmodule MyAppWeb.FallbackController do
  use MyAppWeb, :controller

  def call(conn, {:error, %Ecto.Changeset{} = cs}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(json: MyAppWeb.ChangesetJSON)
    |> render(:error, changeset: cs)
  end

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"404")
  end

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:forbidden)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"403")
  end

  # nil from get_trail/2 means not found (or not authorized)
  def call(conn, nil) do
    conn
    |> put_status(:not_found)
    |> put_view(json: MyAppWeb.ErrorJSON)
    |> render(:"404")
  end
end
```

## JSON Views

```elixir
defmodule MyAppWeb.Api.TrailJSON do
  alias MyApp.Trails.Trail

  def index(%{trails: trails}), do: %{data: Enum.map(trails, &trail/1)}
  def show(%{trail: trail}),    do: %{data: trail(trail)}

  defp trail(%Trail{} = t) do
    %{
      id:         t.id,
      name:       t.name,
      status:     t.status,
      started_at: t.started_at,
      ended_at:   t.ended_at,
      inserted_at: t.inserted_at
    }
  end
end
```

## Router — Dual Auth Pipelines

```elixir
pipeline :api do
  plug :accepts, ["json"]
end

pipeline :api_auth do
  plug :fetch_api_user     # extracts Bearer token
  plug :require_api_user   # 401 if no user
end

# Public endpoints
scope "/api", MyAppWeb.Api do
  pipe_through :api
  post "/auth/login", AuthController, :login
end

# Protected endpoints
scope "/api", MyAppWeb.Api do
  pipe_through [:api, :api_auth]
  resources "/trails", TrailController, only: [:index, :create, :show, :delete]
  post "/trails/:id/pause",    TrailController, :pause
  post "/trails/:id/complete", TrailController, :complete
end
```

## API Auth Plug

```elixir
defmodule MyAppWeb.Accounts.ApiAuth do
  import Plug.Conn

  alias MyApp.Accounts

  def fetch_api_user(conn, _opts) do
    with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
         {:ok, user}          <- Accounts.get_user_by_api_token(token) do
      assign(conn, :current_user, user)
    else
      _ -> assign(conn, :current_user, nil)
    end
  end

  def require_api_user(conn, _opts) do
    if conn.assigns[:current_user] do
      conn
    else
      conn
      |> put_status(:unauthorized)
      |> Phoenix.Controller.json(%{error: "Unauthorized"})
      |> halt()
    end
  end
end
```
