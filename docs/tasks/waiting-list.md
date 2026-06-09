# Your task: activity waiting list

When an activity is full, a member can't register. Build a **waiting list** for full
activities:

- A member can **join** the waiting list of a full activity.
- When a spot frees up — a registered member unregisters — the members on that activity's
  waiting list are **notified**.
- A member **leaves the waiting list only by registering** for the activity. When she
  registers, her waiting-list entry isn't deleted — it's **marked converted** (or whatever
  status you choose). Entries are kept for history.

Build it end to end: backend and frontend.

## The notification

How you deliver the notification is up to you — pick something you can demonstrate working.
For example:

- a stubbed "email" that logs a message (with a link to the activity page) to the console, or
- a websocket push that shows a toast on the activity page in the web app.

The channel doesn't matter; getting the right members notified when a spot opens does.

**The notification must be asynchronous, handled as a side effect** of a spot freeing up —
not done inline in (or blocking) the request that frees the spot.

## Following our conventions

There's no fixed spec for the schema, the GraphQL surface, or the notification plumbing —
that's yours to design. What we care about is that you follow the conventions in the docs.
Read them and apply them:

- `docs/DOMAIN.md` — the data model, and how activities, attendance, and "full" work.
- `docs/ENGINEERING_GUIDE.md` — how we structure backend code, persist data, and run
  background / async work.
- `docs/GRAPHQL.md` — how we expose the API.
- `docs/API.md` — the frontend conventions and how the web app talks to the API.

We're evaluating your judgement and whether the result is correct, tested, and green
against the quality gate — not whether you matched a particular name.
