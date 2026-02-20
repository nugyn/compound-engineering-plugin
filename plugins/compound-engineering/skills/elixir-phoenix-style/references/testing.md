# Testing Patterns

## Test Module Types

| Type | Base module | Use for |
|------|-------------|---------|
| Unit | `MyApp.DataCase` | Context functions, changesets, Ecto queries |
| Controller | `MyAppWeb.ConnCase` | HTTP endpoints, JSON API |
| LiveView | `MyAppWeb.ConnCase` | LiveView mounts, events |
| Unit (pure) | `ExUnit.Case` | Pure Elixir functions, no DB |

## DataCase — Context Tests

```elixir
defmodule MyApp.TrailsTest do
  use MyApp.DataCase

  alias MyApp.Trails

  describe "create_trail/1" do
    test "creates a trail with valid attrs" do
      user = insert(:user)

      assert {:ok, trail} = Trails.create_trail(%{name: "Research", user_id: user.id})
      assert trail.name == "Research"
      assert trail.status == "active"
      assert trail.started_at != nil
    end

    test "returns error changeset with missing name" do
      user = insert(:user)

      assert {:error, changeset} = Trails.create_trail(%{user_id: user.id})
      assert %{name: ["can't be blank"]} = errors_on(changeset)
    end

    test "auto-pauses active trail on create" do
      user = insert(:user)
      active_trail = insert(:trail, user: user, status: "active")

      assert {:ok, _new_trail} = Trails.create_trail(%{name: "New", user_id: user.id})

      paused = Trails.get_trail(active_trail.id, user.id)
      assert paused.status == "paused"
    end
  end

  describe "Trail status transitions" do
    test "active trail can be paused" do
      trail = insert(:trail, status: "active")
      assert {:ok, updated} = Trails.pause_trail(trail)
      assert updated.status == "paused"
    end

    test "completed trail cannot transition" do
      trail = insert(:trail, status: "completed")
      assert {:error, cs} = Trails.pause_trail(trail)
      assert %{status: [_]} = errors_on(cs)
    end
  end
end
```

## ConnCase — API Controller Tests

```elixir
defmodule MyAppWeb.Api.TrailControllerTest do
  use MyAppWeb.ConnCase

  setup %{conn: conn} do
    user  = insert(:user)
    token = MyApp.Accounts.generate_api_token(user)
    conn  = put_req_header(conn, "authorization", "Bearer #{token}")
    %{conn: conn, user: user}
  end

  describe "POST /api/trails" do
    test "creates a trail", %{conn: conn} do
      resp = post(conn, ~p"/api/trails", %{trail: %{name: "Deep Research"}})

      assert %{"data" => %{"id" => _, "name" => "Deep Research", "status" => "active"}} =
               json_response(resp, 201)
    end

    test "returns 422 with invalid params", %{conn: conn} do
      resp = post(conn, ~p"/api/trails", %{trail: %{name: ""}})
      assert %{"errors" => _} = json_response(resp, 422)
    end
  end

  describe "POST /api/trails/:id/pause" do
    test "pauses an active trail", %{conn: conn, user: user} do
      trail = insert(:trail, user: user, status: "active")
      resp  = post(conn, ~p"/api/trails/#{trail.id}/pause")

      assert %{"data" => %{"status" => "paused"}} = json_response(resp, 200)
    end

    test "returns 404 for other user's trail", %{conn: conn} do
      other_trail = insert(:trail, status: "active")  # different user
      resp = post(conn, ~p"/api/trails/#{other_trail.id}/pause")
      assert json_response(resp, 404)
    end
  end
end
```

## LiveView Tests

```elixir
defmodule MyAppWeb.Trails.TrailShowLiveTest do
  use MyAppWeb.ConnCase

  import Phoenix.LiveViewTest

  setup %{conn: conn} do
    user = insert(:user)
    conn = log_in_user(conn, user)
    %{conn: conn, user: user}
  end

  test "renders trail page", %{conn: conn, user: user} do
    trail = insert(:trail, user: user)
    {:ok, _view, html} = live(conn, ~p"/trails/#{trail.id}")

    assert html =~ trail.name
    assert html =~ "active"
  end

  test "pause button transitions trail to paused", %{conn: conn, user: user} do
    trail = insert(:trail, user: user, status: "active")
    {:ok, view, _html} = live(conn, ~p"/trails/#{trail.id}")

    view |> element("button", "Pause") |> render_click()

    assert render(view) =~ "paused"
  end

  test "redirects to trails list for unknown trail", %{conn: conn} do
    assert {:error, {:redirect, %{to: "/trails"}}} = live(conn, ~p"/trails/nonexistent-id")
  end
end
```

## Factories (ExMachina)

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  use ExMachina.Ecto, repo: MyApp.Repo

  def user_factory do
    %MyApp.Accounts.User{
      email: sequence(:email, &"user#{&1}@example.com"),
      hashed_password: Bcrypt.hash_pwd_salt("password123")
    }
  end

  def trail_factory do
    %MyApp.Trails.Trail{
      name: sequence(:trail_name, &"Trail #{&1}"),
      status: "active",
      started_at: DateTime.utc_now(),
      user: build(:user)
    }
  end

  def page_factory do
    %MyApp.Captures.Page{
      url:         sequence(:url, &"https://example.com/page/#{&1}"),
      title:       "Example Page",
      metadata:    %{},
      annotations: [],
      connections: [],
      position:    1,
      captured_at: DateTime.utc_now(),
      trail:       build(:trail)
    }
  end
end
```

## Test Helpers

```elixir
# DataCase provides errors_on/1 for readable changeset assertions
assert %{name: ["can't be blank"], status: ["is invalid"]} = errors_on(changeset)

# ✅ Test ordering with usec precision (never seconds)
trail1 = insert(:trail, inserted_at: ~U[2024-01-01 00:00:00.000001Z])
trail2 = insert(:trail, inserted_at: ~U[2024-01-01 00:00:00.000002Z])
[first | _] = Trails.list_trails(user.id)
assert first.id == trail2.id  # most recent first

# ✅ Async tests safe with DataCase (sandbox mode)
use MyApp.DataCase, async: true
```
