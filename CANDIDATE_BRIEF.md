# Colette — Engineering Take-Home

Welcome, and thanks for taking the time. This page introduces who we are and exactly
what we'd like you to build and review.

## About Colette

[Colette](https://www.colette.club) ([colette.club](https://www.colette.club),
[colette-cohabitation.com](https://colette-cohabitation.com)) is a French company on a
mission to improve daily life for older people and to fight isolation — *créer du lien,
favoriser le partage et les rencontres* ("create bonds, encourage sharing and
encounters"). Solidarity, kindness, and sharing are the values everything is built on.

We do this through two products:

- **Colette Club** — a free activity club for people aged 50+. Members propose, organise
  and join activities every day: sports classes, cultural outings, conferences,
  get-togethers, and online events, across many French cities (around 70,000 members
  today). It's simple, friendly, and commitment-free.
- **Intergenerational cohabitation** — we match seniors (60+) who have a spare room with
  young people (under 30) looking for affordable, quality housing. Hosts gain companionship
  and a little extra income; young renters get a real home (*"less IKEA, more Formica"*).
  We count thousands of hosts.

This exercise models a slice of **Colette Club**: members registering for activities
organised by the club.

## The codebase

The exercise spans our real two-repo stack:

- **`colette-exercise-api`** — Elixir/Phoenix + Absinthe (GraphQL), Ecto/PostGIS,
  `ex_event_bus`, Oban, Dataloader. Most of the work happens here.
  → https://github.com/colette-club/colette-exercise-api
- **`colette-exercise-web`** — React Router v7 (framework mode) + TypeScript, talking to
  the API through a typed GraphQL SDK.
  → https://github.com/colette-club/colette-exercise-web

Start with each repo's **`README.md`** for how to run things — Docker + Postgres/PostGIS
for the API, Bun for the web app — plus the seeded users and the quality gate.

## Your task

### Part 1 — Build the activity waiting list

When an activity is full, a member can't register. Build a **waiting list** for full
activities, **end to end** (backend + frontend):

- A member can **join** the waiting list of a full activity.
- When a spot frees up — a registered member leaves — the members on that activity's
  waiting list are **notified**. The notification must be **asynchronous**, handled as a
  side effect of the seat freeing up — never inline in (or blocking) the request that
  frees it. The channel is your call (a stubbed "email" logged to the console, a websocket
  toast — anything you can demonstrate working).
- A member **leaves the waiting list only by registering** for the activity. Her entry
  isn't deleted — it's **marked converted**. Entries are kept for history.

There's no fixed spec for the schema, the GraphQL surface, or the notification plumbing —
that's yours to design. We're evaluating your **judgement** and whether the result is
correct, tested, and green against the quality gate, not whether you matched a particular
name.

Full task: **`docs/tasks/waiting-list.md`**.

### Part 2 — Review the pull requests

We've opened **two pull requests on GitHub** — *my agenda* and *popular activities* — each
spanning both repos (a `pr/my-agenda` and a `pr/popular-activities` branch, open as PRs in
both [`colette-exercise-api`](https://github.com/colette-club/colette-exercise-api) and
[`colette-exercise-web`](https://github.com/colette-club/colette-exercise-web)).

Each PR already has an automated **GitHub Copilot** review. Go through **both** PRs the way
you'd review a teammate's work and tell us: **in addition to what Copilot flagged, what else
would you change or improve?** Capture it in a **`REVIEW.md`** — for each issue, note *what
it is · how serious it is · why it matters · how you'd fix it* — then fix the most important
two or three.

### Read the docs first — they are the rubric

These encode **our** conventions; match them exactly. They override any default habits,
and your AI assistant will not follow them on its own.

- **`docs/DOMAIN.md`** — the data model: activities, attendance, what "full" means, the
  visibility scope, the geo helper.
- **`docs/ENGINEERING_GUIDE.md`** — backend structure, Ecto and changesets, domain events,
  Oban, testing.
- **`docs/GRAPHQL.md`** — resolvers, auth, errors, Dataloader (no N+1).
- **`docs/API.md`** — the frontend conventions and how the two repos talk.

## Quality gate

There are deliberately **no pre-commit hooks**. Keeping the code formatted, linted,
type-checked and tested is your responsibility — and a graded signal.

```bash
cd colette-exercise-api && make check     # mix format --check-formatted && mix credo --strict && mix test
cd colette-exercise-web && bun run check  # Biome + react-router typegen + tsc + Vitest
```

Both gates must be green before you submit.

## Using AI is encouraged

**We expect you to use an AI assistant — we work this way too.** Use whatever you reach
for (Claude Code, Cursor, Copilot, …); there are no hidden rules against it and no penalty
for it.

But **you own the result**. You are responsible for every line you submit, for steering
the assistant to follow the conventions in `docs/` (which it won't follow by default), and
for the code being correct, tested, and passing the quality gate. We're evaluating your
judgement and the final product — not whether you typed it by hand.

## Submitting

1. **Fork** both repos to your own GitHub account.
2. **Create your own branch** on each fork and do your work there.
3. When you're done, **share the URLs of your forks** with us — no need to open a pull
   request back — and include your **`REVIEW.md`** from Part 2.
4. Make sure **both quality gates are green** before you submit.
