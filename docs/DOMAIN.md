# Domain Model

Colette is a senior activities club. Members register for **activities** organised by
the club. This page documents the schemas as they exist today, plus the feature you'll
build.

All schemas use `binary_id` (UUID) primary and foreign keys, and
`:utc_datetime_usec` timestamps. Aggregate roots (users, activities) are **soft-deleted**
via `archived_at` (see `docs/ENGINEERING_GUIDE.md`).

## Accounts.User

`lib/exercise/accounts/user.ex` — a club member.

| Field         | Type                | Notes |
| ------------- | ------------------- | ----- |
| `id`          | `binary_id`         | PK |
| `email`       | `string`            | **unique** (`validate_required` + `unique_constraint`) |
| `name`        | `string`            | required |
| `archived_at` | `utc_datetime_usec` | soft-delete marker; `nil` = live |
| timestamps    | `utc_datetime_usec` | |

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email, :name, :archived_at])
  |> validate_required([:email, :name])
  |> unique_constraint(:email)
end
```

`Accounts.get_user/1` returns `{:ok, user}` for a non-archived user, else
`{:error, %Exercise.Errors.NotFoundError{}}`. It is what the GraphQL context plug uses to
resolve the bearer token. (`Exercise.Accounts` is a thin facade that delegates to
`Exercise.Accounts.Users`, which owns the query.)

## Communities.Activity

`lib/exercise/communities/activity.ex` — an activity members can attend.

| Field            | Type                          | Notes |
| ---------------- | ----------------------------- | ----- |
| `id`             | `binary_id`                   | PK |
| `title`          | `string`                      | required |
| `slug`           | `string`                      | required, **unique** |
| `description`    | `string` (text)               | |
| `starts_at`      | `utc_datetime_usec`           | required |
| `max_attendees`  | `integer`                     | required, `> 0` |
| `attendee_count` | `integer`                     | **DB-trigger maintained** (count of active attendances); never cast |
| `price_amount`   | `integer`                     | money amount, default `0` |
| `price_currency` | `string`                      | money currency, default `"EUR"` |
| `postal_code`    | `string`                      | |
| `location`       | `Geo.PostGIS.Geometry`        | `geometry(Point, 4326)` (WGS84), GIST-indexed |
| `published_at`   | `utc_datetime_usec`           | `nil` = **draft** |
| `archived_at`    | `utc_datetime_usec`           | soft-delete marker |
| `creator`        | `belongs_to User`             | `on_delete: :nilify_all` |
| `attendances`    | `has_many ActivityAttendance` | |

Money is the pair (`price_amount`, `price_currency`) — there's no single money type;
keep the two fields together.

### Visibility scope

`Communities.Activities.Query.visible/1` is the canonical scope for "members can see
this": published (`published_at` set and in the past) **and** not archived. Query scopes
live in the **query module**, not the schema, and reads go through the
**`Communities.Activities` context** — never the `Repo` from a resolver.

```elixir
# lib/exercise/communities/activities/query.ex — composable, queryable-first scopes
def visible(queryable \\ base()) do
  now = DateTime.utc_now()

  queryable
  |> where([a], not is_nil(a.published_at) and a.published_at <= ^now)
  |> where([a], is_nil(a.archived_at))
end

# lib/exercise/communities/activities.ex — reads go through the context
def list_visible, do: Query.visible() |> Query.order_by_starts_at() |> Repo.all()
def get_visible_by_slug(slug), do: Query.base() |> Query.with_slug(slug) |> Query.visible() |> Repo.one()
```

Use it everywhere you read activities for a member — the `Communities.Activities` getters
(behind the `activities` / `activity` queries and the resolver's `fetch_visible/1`) all go
through it. The web layer calls the `Exercise.Communities` facade, which delegates to
`Communities.Activities` for reads. Follow the same schema / query / context split for
your new schema.

### "Full" — there is no `full?/1` helper

There is **no** `full?/1` function. Fullness is `attendee_count` vs `max_attendees`,
checked in two places:

- **For listing/filtering** (activities with room), use the query scope
  `Communities.Activities.Query.with_available_seats/2`, which compares the
  trigger-maintained `attendee_count` column against `max_attendees` in the database.
- **To enforce capacity on register**, the check lives in `ActivityAttendance`'s
  changeset via `prepare_changes` — so it runs **inside the insert transaction** — and it
  **locks the activity row `FOR UPDATE`** so concurrent registrations serialise and can't
  overbook. `register_to_activity/2` is a thin wrapper that returns the `Repo.insert/2`
  result directly — a full activity comes back as a `:base` (`"activity is full"`)
  changeset error, which the GraphQL `ErrorHandler` normalises into a `ValidationError`
  (see `docs/GRAPHQL.md`). There is no `:activity_full` atom.

```elixir
# lib/exercise/communities/activity_attendance.ex — race-safe capacity guard
defp check_capacity(changeset) do
  prepare_changes(changeset, fn changeset ->
    activity =
      Query.base()
      |> Query.with_id(get_field(changeset, :activity_id))
      |> Query.for_update()              # serialises concurrent registrations
      |> changeset.repo.one()

    if activity && activity.attendee_count >= activity.max_attendees do
      add_error(changeset, :base, "activity is full")
    else
      changeset
    end
  end)
end
```

A read-then-check in the context (without the lock) overbooks under real concurrency —
see `test/exercise/communities/activity_attendances/concurrency_test.exs`.

### Geo helper

`Exercise.Geo` wraps PostGIS. Prefer it over hand-rolled distance math.

```elixir
# lib/exercise/geo.ex
def order_by_distance(query, %Geo.Point{} = point)  # nearest-first ordering
def point(lng, lat)                                 # %Geo.Point{srid: 4326}
```

## Communities.ActivityAttendance

`lib/exercise/communities/activity_attendance.ex` — a member's registration to an
activity. A join row that is **soft-deleted**: leaving stamps `deleted_at` rather than
removing the row (`deleted_at: nil` = an active registration).

| Field        | Type                  | Notes |
| ------------ | --------------------- | ----- |
| `id`         | `binary_id`           | PK |
| `user`       | `belongs_to User`     | `on_delete: :delete_all` |
| `activity`   | `belongs_to Activity` | `on_delete: :delete_all` |
| `deleted_at` | `utc_datetime_usec`   | soft-delete marker; `nil` = active |
| timestamps   | `utc_datetime_usec`   | |

A **partial** unique index on `(user_id, activity_id) WHERE deleted_at IS NULL`
(mirrored by `unique_constraint([:user_id, :activity_id])`) allows only one *active*
registration per member per activity — so a member can re-register after leaving.
`attendance_count/1` and the `attendances` association both scope to `deleted_at IS NULL`.

Created by `Communities.register_to_activity/2` and soft-deleted by
`Communities.unregister_from_activity/2` — these live in the `Communities.ActivityAttendances`
sub-context, exposed through the `Exercise.Communities` facade the web layer calls. They
are the only sanctioned ways to change attendance, and they publish the domain events:

```elixir
# register -> AttendanceCreated
Repo.insert(changeset, success_event: Events.AttendanceCreated, event_opts: [])

# unregister -> AttendanceDeleted (a soft-delete update that frees the seat)
attendance
|> ActivityAttendance.delete_changeset()
|> Repo.update(success_event: Events.AttendanceDeleted, event_opts: [])
```

`AttendanceDeleted` is published whenever a registered member leaves — i.e. **a seat just
freed up**.

---

## Your task

Everything above is the **shared model** the scaffold already ships. On top of it, build
the **[activity waiting list](tasks/waiting-list.md)**: members join the waiting list of a
full activity, are notified when a spot frees up, and leave the list by registering (their
entry is kept, marked converted).

Open the task doc, then build it following the conventions in this guide and in
`docs/ENGINEERING_GUIDE.md`, `docs/GRAPHQL.md`, and `docs/API.md`.
