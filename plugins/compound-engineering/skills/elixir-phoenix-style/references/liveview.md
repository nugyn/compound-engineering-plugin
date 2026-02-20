# LiveView Patterns

## Socket Assigns

```elixir
defmodule MyAppWeb.Trails.TrailShowLive do
  use MyAppWeb, :live_view

  alias MyApp.Trails
  alias MyApp.Captures

  @impl true
  def mount(%{"id" => id}, _session, socket) do
    user = socket.assigns.current_scope.user

    case Trails.get_trail(id, user.id) do
      nil ->
        {:ok, push_navigate(socket, to: ~p"/trails")}

      trail ->
        if connected?(socket) do
          Phoenix.PubSub.subscribe(MyApp.PubSub, "trail:#{trail.id}")
        end

        {:ok,
         socket
         |> assign(:trail, trail)
         |> assign(:pages, Captures.list_pages(trail.id))}
    end
  end

  @impl true
  def handle_params(_params, _uri, socket), do: {:noreply, socket}

  @impl true
  def handle_event("complete_trail", _params, socket) do
    case Trails.complete_trail(socket.assigns.trail) do
      {:ok, trail}    -> {:noreply, assign(socket, :trail, trail)}
      {:error, _cs}   -> {:noreply, put_flash(socket, :error, "Could not complete trail")}
    end
  end

  @impl true
  def handle_info({:page_captured, page}, socket) do
    {:noreply, update(socket, :pages, fn pages -> [page | pages] end)}
  end
end
```

## Forms

```elixir
# In mount/3 or handle_params/3, initialize with to_form/1:
|> assign(:form, to_form(Trails.change_trail(%Trail{})))

# In handle_event for validation:
def handle_event("validate", %{"trail" => params}, socket) do
  form =
    %Trail{}
    |> Trails.change_trail(params)
    |> Map.put(:action, :validate)
    |> to_form()

  {:noreply, assign(socket, :form, form)}
end

# In handle_event for save:
def handle_event("save", %{"trail" => params}, socket) do
  case Trails.create_trail(params) do
    {:ok, trail} ->
      {:noreply, push_navigate(socket, to: ~p"/trails/#{trail}")}

    {:error, changeset} ->
      {:noreply, assign(socket, :form, to_form(changeset))}
  end
end
```

## Components (Function Components)

```elixir
# In core_components.ex or a feature-specific component module
attr :trail, :map, required: true
attr :class, :string, default: ""

def trail_card(assigns) do
  ~H"""
  <div class={["card bg-base-100 shadow-sm", @class]}>
    <div class="card-body">
      <h2 class="card-title">{@trail.name}</h2>
      <.status_badge status={@trail.status} />
    </div>
  </div>
  """
end

attr :status, :string, required: true

def status_badge(assigns) do
  ~H"""
  <span class={[
    "badge",
    @status == "active" && "badge-success",
    @status == "paused" && "badge-warning",
    @status == "completed" && "badge-neutral"
  ]}>
    {@status}
  </span>
  """
end
```

## LiveView Do/Don't

```elixir
# ✅ Delegate to contexts
def handle_event("pause", _params, socket) do
  case Trails.pause_trail(socket.assigns.trail) do
    {:ok, trail} -> {:noreply, assign(socket, :trail, trail)}
    {:error, _}  -> {:noreply, put_flash(socket, :error, "Cannot pause")}
  end
end

# ❌ Don't call Repo directly in LiveView
def handle_event("pause", _params, socket) do
  trail = socket.assigns.trail |> Ecto.Changeset.change(%{status: "paused"}) |> Repo.update!()
  {:noreply, assign(socket, :trail, trail)}
end

# ✅ Subscribe to PubSub in mount (only when connected)
if connected?(socket) do
  Phoenix.PubSub.subscribe(MyApp.PubSub, "trail:#{trail.id}")
end

# ✅ Use push_navigate for cross-LiveView navigation (full mount)
# ✅ Use push_patch for same LiveView param changes (handle_params only)

# ✅ assign_new for expensive or conditional assigns
|> assign_new(:pages, fn -> Captures.list_pages(trail.id) end)
```

## LiveView Auth

```elixir
# In router.ex — scope requires auth with on_mount hook
live_session :require_authenticated,
  on_mount: [{MyAppWeb.Accounts.UserAuth, :require_authenticated}],
  layout: {MyAppWeb.Layouts, :app} do
  live "/trails", Trails.TrailListLive, :index
  live "/trails/:id", Trails.TrailShowLive, :show
end

# Access user via socket.assigns.current_scope
def mount(_params, _session, socket) do
  user = socket.assigns.current_scope.user
  ...
end
```
