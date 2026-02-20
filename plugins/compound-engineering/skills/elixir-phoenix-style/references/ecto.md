# Ecto Patterns

## Schemas

```elixir
defmodule MyApp.Trails.Trail do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id

  schema "trails" do
    field :name, :string
    field :status, :string, default: "active"
    field :started_at, :utc_datetime_usec  # ← ALWAYS usec, never :utc_datetime
    field :ended_at,   :utc_datetime_usec

    belongs_to :user, MyApp.Accounts.User

    timestamps(type: :utc_datetime_usec)  # ← ALWAYS usec
  end
end
```

**Schema rules:**
- Binary UUIDs: `@primary_key {:id, :binary_id, autogenerate: true}` + `@foreign_key_type :binary_id`
- Timestamps: always `type: :utc_datetime_usec` — second precision causes ordering bugs in tests
- JSONB fields: `field :metadata, :map` / `field :tags, {:array, :map}`
- After any insert/update of a JSONB record, reload from DB — Postgres normalizes keys to strings

## Changesets

**One changeset per operation, not one generic changeset:**

```elixir
# ✅ Purpose-named changesets
def create_changeset(attrs) do
  %__MODULE__{}
  |> cast(attrs, [:name, :user_id])
  |> validate_required([:name, :user_id])
  |> validate_length(:name, max: 255)
  |> put_change(:status, "active")
  |> put_change(:started_at, DateTime.utc_now())
end

def status_changeset(trail, new_status) do
  trail
  |> change()
  |> put_change(:status, new_status)
  |> validate_status_transition()
end

def update_changeset(trail, attrs) do
  trail
  |> cast(attrs, [:name])
  |> validate_required([:name])
  |> validate_length(:name, max: 255)
end
```

**State machine validation in changeset:**

```elixir
defp validate_status_transition(changeset) do
  current = changeset.data.status
  next    = get_change(changeset, :status)

  valid? = case {current, next} do
    {"active",    "paused"}    -> true
    {"active",    "completed"} -> true
    {"paused",    "active"}    -> true
    {"paused",    "completed"} -> true
    {"completed", _}           -> false
    {same, same}               -> true
    _                          -> false
  end

  if valid?, do: changeset, else: add_error(changeset, :status, "invalid transition from #{current} to #{next}")
end
```

## Transactions — Ecto.Multi

Use `Ecto.Multi` when multiple writes must be atomic:

```elixir
def create_trail(attrs) do
  user_id   = attrs[:user_id] || attrs["user_id"]
  changeset = Trail.create_changeset(attrs)

  if changeset.valid? do
    Repo.transaction(fn ->
      # 1. Auto-pause any currently active trail
      case get_active_trail(user_id) do
        nil -> :ok
        active ->
          case pause_trail(active) do
            {:ok, _}    -> :ok
            {:error, cs} -> Repo.rollback(cs)
          end
      end

      # 2. Insert the new trail
      case Repo.insert(changeset) do
        {:ok, trail} -> trail
        {:error, cs} -> Repo.rollback(cs)
      end
    end)
  else
    {:error, changeset}
  end
end
```

Or with `Ecto.Multi` for readability on more complex flows:

```elixir
Ecto.Multi.new()
|> Ecto.Multi.insert(:trail, Trail.create_changeset(attrs))
|> Ecto.Multi.run(:pause_active, fn _repo, _changes ->
  pause_active_trail_for_user(user_id)
end)
|> Repo.transaction()
```

## Queries

```elixir
import Ecto.Query

# ✅ Scoped to user (authorization built in)
def list_trails(user_id) do
  Trail
  |> where([t], t.user_id == ^user_id)
  |> order_by([t], desc: t.inserted_at)
  |> Repo.all()
end

# ✅ Safe get (returns nil, doesn't raise)
def get_trail(id, user_id) do
  Repo.get_by(Trail, id: id, user_id: user_id)
end

# ✅ Preloading associations
def get_trail_with_pages(id, user_id) do
  Trail
  |> where([t], t.id == ^id and t.user_id == ^user_id)
  |> preload(:pages)
  |> Repo.one()
end

# ✅ Composable query fragments
def active_scope(query \\ Trail) do
  where(query, [t], t.status == "active")
end

def for_user(query, user_id) do
  where(query, [t], t.user_id == ^user_id)
end

# Usage:
Trail |> active_scope() |> for_user(user_id) |> Repo.all()
```

## Migrations

```elixir
# ✅ Binary UUID primary key
create table(:trails, primary_key: false) do
  add :id, :binary_id, primary_key: true
  add :name, :string, null: false
  add :status, :string, null: false, default: "active"
  add :user_id, references(:users, type: :binary_id, on_delete: :delete_all), null: false
  add :started_at, :utc_datetime_usec
  add :ended_at, :utc_datetime_usec
  timestamps(type: :utc_datetime_usec)
end

create index(:trails, [:user_id])
create index(:trails, [:user_id, :status])  # for active trail lookups

# ✅ Changing column type safely
alter table(:trails) do
  modify :inserted_at, :utc_datetime_usec, from: :utc_datetime
  modify :updated_at,  :utc_datetime_usec, from: :utc_datetime
end
```

## JSONB Gotcha

```elixir
# ❌ Atom keys from changeset won't match string keys after DB round-trip
page = Repo.get!(Page, page.id)
page.annotations  # => [%{"text" => "hello"}]  ← string keys from Postgres

# ✅ Always reload after insert to get normalized keys
{:ok, page} = Repo.insert(changeset)
page = Repo.get!(Page, page.id)  # reload with Postgres-normalized keys
```
