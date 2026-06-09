# Engineering Guide

How we build the Colette backend. These conventions are **enforced** (by `mix credo
--strict`, by code review, and by the test suite). Read this before writing any code.
The patterns below are not stylistic suggestions — match them exactly.

## Architecture: context-based design

All database access lives in **contexts**. A context is a plain module under
`lib/exercise/` that owns a slice of the domain and is the *only* place allowed to
touch the `Repo` for that slice.

- `Exercise.Accounts` — users (a facade delegating to `Exercise.Accounts.Users`).
- `Exercise.Communities` — the facade the web layer calls. It delegates to two
  sub-contexts:
  - `Exercise.Communities.Activities` — activity **reads** (`list_visible/0`,
    `get_visible/1`, `get_visible_by_slug/1`), backed by `Communities.Activities.Query`.
  - `Exercise.Communities.ActivityAttendances` — attendance **writes** and counts
    (`register_to_activity/2`, `unregister_from_activity/2`, `attendance_count/1`),
    backed by `Communities.ActivityAttendances.Query`.

Web code (resolvers, plugs, the schema) **never queries the `Repo` directly** for
business reads/writes. It calls context functions (through the `Exercise.Communities`
facade), which return `{:ok, _}` / `{:error, _}` tuples.

```elixir
# lib/exercise/communities/activity_attendances.ex
def register_to_activity(user, activity) do
  # Capacity is enforced inside the changeset (see ActivityAttendance.check_capacity);
  # the context stays a thin wrapper that publishes the event and returns the result
  # directly. A full activity comes back as a `:base` ("activity is full") changeset
  # error and a duplicate as a unique-constraint error — the web ErrorHandler turns
  # either into a ValidationError. No mapping happens here.
  %ActivityAttendance{}
  |> ActivityAttendance.changeset(%{user_id: user.id, activity_id: activity.id})
  |> Repo.insert(success_event: Events.AttendanceCreated, event_opts: [])
end
```

Reads go through a context too. Activity reads live in `Exercise.Communities.Activities`
(`list_visible/0`, `get_visible/1`, `get_visible_by_slug/1`), which compose scopes from
`Exercise.Communities.Activities.Query` — resolvers call the `Exercise.Communities`
facade and never build queries or call `Repo`.

```elixir
# lib/exercise_web/resolvers/activities.ex — reads via the context, not Repo
defp fetch_visible(id) do
  case Communities.get_visible(id) do
    nil -> {:error, Errors.NotFoundError.new(type: "Activity", fields: %{id: id})}
    activity -> {:ok, activity}
  end
end
```

The `NotFoundError` struct is the web layer shaping a context `nil` into a GraphQL error —
that mapping is the resolver's job; the data access is the context's. Errors are
`Exercise.Errors` structs, never bare maps or atoms (see `docs/GRAPHQL.md`). Business
logic (the "is it full?" check, the write, the event) stays in the context.

## Soft deletes

We **never hard-delete**. Aggregates (users, activities) soft-delete via an
`archived_at` timestamp; join rows (e.g. `ActivityAttendance`) soft-delete via
`deleted_at`. A `nil` marker means "live".

- Schemas carry the marker (`archived_at` / `deleted_at`) as `:utc_datetime_usec`.
- Read paths filter it out. `Accounts.get_user/1` only returns non-archived users;
  `Communities.Activities.Query.visible/1` excludes archived activities;
  `attendance_count/1` and the `attendances` association exclude soft-deleted rows.
- Leaving an activity stamps `deleted_at` (keeping the row) and publishes
  `AttendanceDeleted` via a `Repo.update` — see `Communities.unregister_from_activity/2`.

```elixir
# lib/exercise/accounts/users.ex (Exercise.Accounts delegates here)
def get_user(id) do
  Query.active()            # scopes to is_nil(archived_at)
  |> Repo.get(id)
  |> case do
    nil -> {:error, Errors.NotFoundError.new(type: "User", fields: %{id: id})}
    user -> {:ok, user}
  end
end
```

Note: soft-deleting an `ActivityAttendance` is what **frees a seat** — the DB trigger
recomputes `attendee_count` on the `deleted_at` update, and `AttendanceDeleted` is
published in the same transaction. We still never hard-delete: the row stays for history,
and the partial unique index
(`WHERE deleted_at IS NULL`) lets a member re-register after leaving.

## Elixir style (enforced by `mix credo --strict`)

These four rules trip up most newcomers. Credo will fail the build on the first three;
the fourth is enforced in review.

### 1. `alias` and module attributes at the TOP only

Every `alias` and every module attribute (`@foo value`) lives in a block at the top of
the module — never inline, never mid-module, never inside a function.

```elixir
# GOOD
defmodule Exercise.Communities.Activities do
  @moduledoc "The Activities context: activity reads and attendance writes."
  import Ecto.Query

  alias Exercise.Communities.Activities.Query
  alias Exercise.Communities.Activity
  alias Exercise.Communities.ActivityAttendance
  alias Exercise.Communities.Events
  alias Exercise.Repo

  # ... functions ...
end
```

```elixir
# BAD — alias mid-module / inside a function
defmodule Exercise.Communities.Activities do
  def register_to_activity(user, activity) do
    alias Exercise.Communities.Activity  # ❌ never do this
    # ...
  end
end
```

### 2. No single-value pipes

Do not pipe a lone value into a single function. Call the function directly. Pipes are
for **multi-step** transformations only.

```elixir
# BAD
attendance_count |> length()
user.id |> to_string()

# GOOD
length(attendance_count)
to_string(user.id)

# GOOD — multi-step pipe is fine
ActivityAttendance
|> where([a], a.activity_id == ^id)
|> Repo.aggregate(:count)
```

### 3. Every module needs a `@moduledoc`

No exceptions for application modules. Even one-liners:

```elixir
defmodule Exercise.Geo do
  @moduledoc "Geospatial helpers. Prefer these over hand-rolled distance math."
  # ...
end
```

### 4. `binary_id` (UUID) primary keys everywhere

Every schema uses UUID primary keys and UUID foreign keys. Generators are already
configured for this (`config :exercise, generators: [timestamp_type: :utc_datetime, binary_id: true]`),
and every migration adds `:id, :binary_id, primary_key: true`.

```elixir
# Every schema starts like this
@primary_key {:id, :binary_id, autogenerate: true}
@foreign_key_type :binary_id
schema "activities" do
  # ...
end
```

```elixir
# Every migration
create table(:activities, primary_key: false) do
  add :id, :binary_id, primary_key: true
  add :creator_id, references(:users, type: :binary_id, on_delete: :nilify_all)
  # ...
end
```

## Changesets: validate in the changeset

Validation belongs in the schema's `changeset/2` — `validate_required`,
`validate_number`, `unique_constraint`, `assoc_constraint`. Contexts and resolvers do
not re-validate; they trust the changeset and surface its errors.

```elixir
# lib/exercise/communities/activity.ex
def changeset(activity, attrs) do
  activity
  |> cast(attrs, [
    :title, :slug, :description, :starts_at, :max_attendees,
    :price_amount, :price_currency, :postal_code, :location,
    :published_at, :archived_at, :creator_id
  ])
  |> validate_required([:title, :slug, :starts_at, :max_attendees])
  |> validate_number(:max_attendees, greater_than: 0)
  |> unique_constraint(:slug)
end
```

```elixir
# lib/exercise/communities/activity_attendance.ex
def changeset(attendance, attrs) do
  attendance
  |> cast(attrs, [:user_id, :activity_id])
  |> validate_required([:user_id, :activity_id])
  |> unique_constraint([:user_id, :activity_id])
  |> assoc_constraint(:user)
  |> assoc_constraint(:activity)
  |> check_capacity()   # prepare_changes + FOR UPDATE lock; see docs/DOMAIN.md
end
```

The `unique_constraint([:user_id, :activity_id])` is backed by a DB unique index —
that's what makes a double registration fail cleanly with a changeset error instead of
a raw Postgres exception. **New tables follow the same pattern**: a DB constraint plus
a matching `unique_constraint` in the changeset.

## Domain events (`ex_event_bus`)

Domain events are published by the **`Repo` wrapper on successful writes**, via the
`success_event:` option. You do not call an event bus by hand after a write — you pass
the event to `Repo.insert/2`, `Repo.update/2`, or `Repo.delete/2` and it fires only if
the write succeeds (and within the same transaction).

```elixir
# lib/exercise/repo.ex
defmodule Exercise.Repo do
  use Ecto.Repo, otp_app: :exercise, adapter: Ecto.Adapters.Postgres
  use ExEventBus.EctoRepoWrapper, ex_event_bus: Exercise.EventBus
end
```

```elixir
# Publish on insert
Repo.insert(changeset, success_event: Events.AttendanceCreated, event_opts: [])

# Publish on a soft-delete update (leaving an activity frees a seat)
attendance
|> ActivityAttendance.delete_changeset()
|> Repo.update(success_event: Events.AttendanceDeleted, event_opts: [])
```

Events are plain structs declared with `defevents/1`:

```elixir
# lib/exercise/communities/events.ex
defmodule Exercise.Communities.Events do
  @moduledoc "Community domain events, published by the Repo wrapper on successful writes."
  use ExEventBus.Event

  defevents([
    AttendanceCreated,
    AttendanceDeleted
  ])
end
```

Each event struct carries `aggregate` / `changes` / `metadata` fields. `aggregate` is
the record that was written (e.g. the `ActivityAttendance` behind an `AttendanceCreated`
or `AttendanceDeleted`) — that's how a handler knows *which* record changed.

### Handlers

A handler subscribes to fully-qualified event names and implements `handle_event/1`,
which returns `:ok`. Handlers run **asynchronously as Oban jobs** on the event bus's
own Oban instance (queue `:ex_event_bus`) — so they're where you kick off a side effect
in reaction to a write, without blocking it.

```elixir
defmodule Exercise.Communities.SomeEventHandler do
  @moduledoc "Reacts to a domain event and kicks off follow-up work."
  use ExEventBus.EventHandler,
    ex_event_bus: Exercise.EventBus,
    events: ["Elixir.Exercise.Communities.Events.SomeEvent"]

  def handle_event(%{aggregate: aggregate}) do
    # ... react to the written record — e.g. enqueue an Oban worker for the slow part ...
    :ok
  end
end
```

## Background jobs (Oban)

The application has **its own Oban instance**, separate from the event bus's. Use it
for application workers. It has one queue, `:notifications`.

```elixir
# config/config.exs
config :exercise, Oban,
  engine: Oban.Engines.Basic,
  notifier: Oban.Notifiers.PG,
  repo: Exercise.Repo,
  queues: [notifications: 10]
```

```elixir
defmodule Exercise.Communities.SomeWorker do
  @moduledoc "Performs one unit of async work off the :notifications queue."
  use Oban.Worker, queue: :notifications

  @impl Oban.Worker
  def perform(%Oban.Job{args: args}) do
    # ... do the work (deliver something, call an external service, ...) ...
    :ok
  end
end
```

Enqueue with `Oban.insert/1`:

```elixir
%{some_id: record.id}
|> Exercise.Communities.SomeWorker.new()
|> Oban.insert()
```

> Two Oban instances: the **event bus** uses queue `:ex_event_bus` to dispatch domain
> events to handlers; the **application** uses queue `:notifications` for your workers.
> Don't enqueue application jobs on the event-bus instance.

## Testing

We use **ExUnit** with **ExMachina** factories (`test/support/factory.ex`). All tests
that touch the DB run inside the Ecto SQL sandbox (`Exercise.DataCase`); most are
`async: true`.

### Context tests

```elixir
defmodule Exercise.Communities.ActivityAttendancesTest do
  use Exercise.DataCase, async: true

  import Exercise.Factory

  alias Exercise.Communities

  test "creates an attendance" do
    user = insert(:user)
    activity = insert(:activity, max_attendees: 5)

    assert {:ok, _attendance} = Communities.register_to_activity(user, activity)
    assert Communities.attendance_count(activity) == 1
  end
end
```

`errors_on/1` (from `DataCase`) turns changeset errors into a map for assertions:

```elixir
assert {:error, changeset} = Communities.register_to_activity(user, activity)
assert %{user_id: ["has already been taken"]} = errors_on(changeset)
```

### Events and Oban in tests

In the test environment both Oban instances are `testing: :manual` — jobs and events
are **enqueued but not run automatically**. Drive them explicitly.

- **Events**: `use ExEventBus.Testing, ex_event_bus: Exercise.EventBus` injects
  `assert_event_received/1` (the event was enqueued on the bus) and `execute_events/0`
  (drain the `:ex_event_bus` queue, running the handlers synchronously).
- **Oban jobs**: `Oban.Testing.assert_enqueued(worker: MyWorker, args: %{...})`.

```elixir
use ExEventBus.Testing, ex_event_bus: Exercise.EventBus

test "a write publishes an event, whose handler enqueues a worker" do
  # ... do a write that publishes SomeEvent ...

  assert_event_received(Exercise.Communities.Events.SomeEvent)
  execute_events()  # drains the :ex_event_bus queue, running the handler

  assert_enqueued(worker: Exercise.Communities.SomeWorker)
end
```

See `docs/GRAPHQL.md` for the GraphQL test helper (`run/2` in `ExerciseWeb.GqlCase`).

## Quality gate

There are **deliberately no pre-commit hooks**. Keeping the code formatted, linted, and
green is your responsibility — and a graded signal.

```bash
make check   # mix format --check-formatted && mix credo --strict && mix test
```

Run it before you consider any change done. If you change a file with a pre-existing
credo issue in scope, fix it.
