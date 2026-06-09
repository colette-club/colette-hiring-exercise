# GraphQL

The backend exposes a single GraphQL endpoint built with **Absinthe**. The current SDL
lives in `priv/schema.graphql` (regenerate/inspect it from the playground). This page
is how we write the *backend* side; for the frontend's typed client see `docs/API.md`.

- Endpoint: `POST /api`
- Playground: `GET /graphiql` (GraphiQL, `interface: :playground`)

## Layout

| Concern        | File |
| -------------- | ---- |
| Schema root    | `lib/exercise_web/schema.ex` |
| Object/input types | `lib/exercise_web/schema/types.ex` |
| Resolvers      | `lib/exercise_web/resolvers/` |
| Middleware     | `lib/exercise_web/schema/middleware/` |
| Context plug   | `lib/exercise_web/context.ex` |

## Resolvers

A resolver takes `(parent, args, resolution)` and returns `{:ok, _}` or `{:error, _}`.
On protected fields you destructure the current user straight out of the context.

```elixir
# lib/exercise_web/resolvers/activities.ex
def register(_parent, %{input: %{activity_id: id}}, %{context: %{current_user: user}}) do
  with {:ok, activity} <- fetch_visible(id),
       {:ok, attendance} <- Communities.register_to_activity(user, activity) do
    {:ok, %{attendance: attendance}}
  end
end
```

Resolvers stay thin: resolve a visibility scope, delegate to a context, wrap the result
in the payload shape. **No business logic, no `Repo` writes in resolvers** — see the
context rule in `docs/ENGINEERING_GUIDE.md`.

## Wiring a field

Fields are declared in `schema.ex` (queries/mutations) and `types.ex` (objects/inputs).
Mutations follow an **input-object + payload** shape.

```elixir
# lib/exercise_web/schema.ex
mutation do
  @desc "Register the current member to an activity."
  field :register_to_activity, :register_to_activity_payload do
    arg(:input, non_null(:register_to_activity_input))
    middleware(Authenticated)
    resolve(&Resolvers.Activities.register/3)
    middleware(ErrorHandler)
  end
end
```

```elixir
# lib/exercise_web/schema/types.ex
input_object :register_to_activity_input do
  field :activity_id, non_null(:id)
end

object :register_to_activity_payload do
  field :attendance, :activity_attendance
end
```

Note the **middleware order**: `Authenticated` runs *before* the resolver,
`ErrorHandler` runs *after*. That ordering is load-bearing (see below).

## Authentication

For this exercise the bearer token **is the user id** — there is no JWT. Send:

```
Authorization: Bearer <user-id>
```

The context plug resolves it to `current_user`. Missing/invalid token simply means no
`current_user` in context (the request still runs; protected fields then reject).

```elixir
# lib/exercise_web/context.ex
def call(conn, _opts) do
  with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
       {:ok, user} <- Accounts.get_user(token) do
    Absinthe.Plug.put_options(conn, context: %{current_user: user})
  else
    _ -> conn
  end
end
```

Protect a field by placing the `Authenticated` middleware **before** the resolver. It
rejects when there's no `current_user`:

```elixir
# lib/exercise_web/schema/middleware/authenticated.ex
# alias Exercise.Errors
def call(%{context: %{current_user: %_{}}} = resolution, _config), do: resolution

def call(resolution, _config) do
  Absinthe.Resolution.put_result(
    resolution,
    {:error, Errors.UnauthenticatedError.new()}
  )
end
```

The error is an `Exercise.Errors` struct — `ErrorHandler` (below) renders it with
`extensions.errorCode == "UnauthenticatedError"`.

## Errors

Resolvers and contexts return either a `%Ecto.Changeset{}` or an **`Exercise.Errors`
struct** (an "ex_error") — never a bare atom or a hand-built map. The **`ErrorHandler`
middleware**, placed *after* the resolver, turns those into GraphQL errors: it expands a
changeset into `ValidationError`s and renders every ex_error with its `error_code` (plus
any extra fields) under `extensions`.

```elixir
# lib/exercise_web/schema/middleware/error_handler.ex (note the spelling: normalize/1)
defp normalize({:error, error}) when is_ex_error(error), do: handle(error)
defp normalize({:error, %Changeset{} = changeset}), do: handle(changeset)
defp normalize(other), do: handle(other)

# A changeset becomes one ValidationError per field; anything unrecognised becomes an
# UnhandledError. Each ex_error is then rendered as, e.g.:
#   %{message: "...", extensions: %{"errorCode" => "ValidationError", "reasons" => [...]}}
```

Domain errors are `Exercise.Errors.*` structs defined with `defexerror`:

```elixir
# lib/exercise/errors/not_full_error.ex
defmodule Exercise.Errors.NotFullError do
  @moduledoc "Returned when joining the waiting list of an activity that isn't full."
  use Exercise.ExErrors

  defexerror([:activity_id, message: "activity is not full"], required_fields: [:activity_id])
end
```

So to add a new domain error: **define an `Exercise.Errors.*` module and return
`{:error, Errors.NotFullError.new(activity_id: id)}` from your context.** You do **not**
touch `ErrorHandler` — its `is_ex_error` clause already handles any ex_error, surfacing it
as `extensions.errorCode == "NotFullError"` (the `error_code` defaults to the module's
last name segment).

A full activity is *not* a domain atom: `register_to_activity/2` returns the changeset's
`:base` (`"activity is full"`) error, which `ErrorHandler` turns into a `ValidationError`
with `reasons: ["activity is full"]`. Don't build GraphQL error maps inside resolvers.

## Avoiding N+1: Dataloader

Associations are loaded with **Dataloader**, never per-row in a resolver. The loader is
built once per request in `Schema.context/1` from a single `GenericEcto` source
(`ExerciseWeb.Dataloaders.GenericEcto`, which wraps `Dataloader.Ecto` and adds
query-function and count batches).

```elixir
# lib/exercise_web/schema.ex
def context(ctx) do
  loader =
    Dataloader.new()
    |> Dataloader.add_source(GenericEcto, GenericEcto.data())

  Map.put(ctx, :loader, loader)
end

def plugins do
  [Absinthe.Middleware.Dataloader] ++ Absinthe.Plugin.defaults()
end
```

For a plain association, use the `dataloader/1` helper:

```elixir
# lib/exercise_web/schema/types.ex
field :creator, :user, resolve: dataloader(GenericEcto)
field :participants, non_null(list_of(non_null(:user))), resolve: dataloader(GenericEcto)
```

### Viewer-scoped association (no N+1)

A "current viewer's …" field still goes through the loader — pass a scoped `query_fun` in
the batch key so the whole list batches into one query. **This is the exact pattern a
viewer-scoped field (the current viewer's entry for an activity) should follow:**

```elixir
# lib/exercise_web/resolvers/activities.ex — Activity.viewerIsRegistered
# alias Exercise.Communities.ActivityAttendances.Query, as: AttendanceQuery
def viewer_is_registered(activity, _args, %{
      context: %{current_user: %{id: user_id}, loader: loader}
    }) do
  batch_key = {:attendances, %{query_fun: {&AttendanceQuery.with_user_id/2, user_id}}}

  loader
  |> Dataloader.load(GenericEcto, batch_key, activity)
  |> on_load(fn loaded ->
    {:ok, Dataloader.get(loaded, GenericEcto, batch_key, activity) != []}
  end)
end

# No signed-in viewer → don't load anything.
def viewer_is_registered(_activity, _args, _resolution), do: {:ok, false}
```

### Stored aggregate — read the column

For a **stored aggregate**, read the column; don't count per row. `attendanceCount` reads
the trigger-maintained `attendee_count` (see `docs/DOMAIN.md`), and `remainingSpots`
derives from it:

```elixir
field :attendance_count, non_null(:integer) do
  resolve(fn activity, _args, _resolution -> {:ok, activity.attendee_count} end)
end

field :remaining_spots, non_null(:integer) do
  resolve(fn activity, _args, _resolution ->
    {:ok, max(activity.max_attendees - activity.attendee_count, 0)}
  end)
end
```

### On-the-fly aggregate — batch with on_load

An aggregate with **no** stored column must be **batched**, not computed per row. Load the
association once through the loader and reduce it in `on_load` — unlike `attendanceCount`,
which just reads its column:

```elixir
field :some_aggregate, :float do
  resolve(fn record, _args, %{context: %{loader: loader}} ->
    loader
    |> Dataloader.load(GenericEcto, :some_assoc, record)
    |> on_load(fn loaded ->
      rows = Dataloader.get(loaded, GenericEcto, :some_assoc, record)
      {:ok, summarise(rows)}
    end)
  end)
end
```

> **Never** load an association by calling the `Repo` (or `Repo.preload`) inside a
> per-row resolver. New association/aggregate fields use `dataloader/1` or an `on_load`
> batch like the ones above. A new field that does a query-per-row will fail review.

## Testing GraphQL

GraphQL tests `use ExerciseWeb.ConnCase` **and** `use ExerciseWeb.GqlCase` (the
`gql_case` package). Put the operation in a `.gql` file **next to the test**, load it with
`load_gql_file/1`, then run it with `query_gql/1`: `:current_user` authenticates as a user
(sends `Bearer <id>`); `:variables` passes variables. `query_gql/1` returns the full
decoded response (`data["data"]` / `data["errors"]`).

```elixir
defmodule ExerciseWeb.Schema.Mutations.RegisterToActivityTest do
  use ExerciseWeb.ConnCase, async: true
  use ExerciseWeb.GqlCase

  import Exercise.Factory

  load_gql_file("RegisterToActivity.gql")   # sits beside this test file

  test "registering a full activity returns a ValidationError" do
    activity = insert(:activity, max_attendees: 1)
    insert(:activity_attendance, activity: activity)

    data = query_gql(current_user: insert(:user), variables: %{id: activity.id})

    assert [
             %{"extensions" => %{"errorCode" => "ValidationError", "reasons" => ["activity is full"]}}
           ] = data["errors"]
  end
end
```

Without `:current_user` the request is unauthenticated, and protected mutations return an
`UnauthenticatedError` (assert `data["errors"]` carries
`extensions.errorCode == "UnauthenticatedError"`) — use that for auth coverage.
