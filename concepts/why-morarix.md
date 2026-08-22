# Why Morarix

## The problem, in one scenario

Say you're building a checkout service. It's not exotic — it depends on
Postgres for orders, Redis for cart/session state, and S3 for storing
receipts. Three dependencies, all boring, all standard.

Here's what actually happens when a new engineer joins the team and tries
to run it locally:

- They clone the repo, run the app, and it crashes — no Postgres running.
- They find a `docker-compose.yml`, maybe. Maybe it's current. Maybe it's
  the one from eight months ago, before someone added Redis by hand and
  never wrote it down.
- They get Postgres up, but the schema's wrong, because migrations live
  in three different places and nobody's sure which one is authoritative
  anymore.
- Eventually it runs. Then they try to write a test, and the test suite
  is half real-Postgres integration tests that only pass if you happen
  to have the right container running, and half mocked-everything unit
  tests that pass regardless of whether the real Redis client would
  actually work.
- In CI, none of this matches local. CI has its own hand-maintained
  YAML, drifting from the compose file, drifting from what's in someone's
  head.

None of this is a single bug. It's accumulated drift between "what the
service actually depends on" and "what's written down anywhere" — and it
gets worse with every dependency added, because nothing forces the two to
stay in sync.

## What Morarix actually does

Morarix's answer is narrow on purpose: **there is exactly one place the
dependency graph is declared** — `morarix.graph.json` — and everything
else (local dev, CI, a new hire's first `git clone`) reads from that one
place instead of reconstructing it independently.

For the checkout service above, that graph says: Postgres, real, this
image and version; Redis, real; S3, real, backed by LocalStack, bucket
name `receipts`. One file, git-committed, human-readable, no secrets in
it.

`morarix up` reads that file and does the boring, mechanical part:
provisions containers for whatever's declared `real` (never builds an
image — every real container is a pull from a public registry, the
checkout service itself still runs as your normal local process), waits
for each one to actually be healthy — not "container started," but
"Postgres will accept a connection" — and then writes the connection
info to `.env.morarix`. Then it gets out of the way. Your tests run with
your existing framework, exactly the same command locally and in CI:

```
morarix up
go test ./...   # or npm test, pytest, mvn test, dotnet test, rspec
morarix down
```

That's the whole interaction. Morarix doesn't run your tests, doesn't
touch your test code, doesn't care what language or framework you use.
It solves the one problem underneath all the friction above: making sure
"what's declared" and "what's actually running" are the same thing,
everywhere, every time.

## What this buys you

Going back to the new-engineer scenario: with a graph in place, onboarding
is `morarix init` (or `morarix import compose` if a `docker-compose.yml`
already exists) once, ever, followed by `morarix up` every time after
that. The schema question resolves itself, because `seed_files` — if
declared — runs once, from a clean container, every run; there's no
"which migration is current" ambiguity because there's no accumulated
state to be ambiguous about.

The mocked-vs-real test gap narrows too, not because Morarix generates
tests (it doesn't — see the [testing philosophy](../docs/usage-guide.md)
section of the usage guide for how that's meant to work), but because
running against a real, healthy dependency stops being the expensive,
fragile option. It's one command, and it's the same command your CI
already runs.

And CI drift stops being possible by construction: CI runs `morarix up`
against the same `morarix.graph.json` a laptop does. There's no second
copy of the dependency list to fall out of sync.

## What Morarix deliberately doesn't do

- It doesn't infer your dependencies. `morarix.graph.json` is always
  hand-authored — scaffolded by `morarix init` or converted once from an
  existing compose file, then reviewed by a person. Autodetection that's
  wrong is worse than no autodetection, especially for something as
  consequential as "does this need to be a real container."
- It doesn't build images. Every `real` dependency is a public registry
  pull; the service you're actually working on never runs inside
  Morarix's control.
- It doesn't write or run your tests. It hands off connection info and
  stops. What you do with a healthy Postgres container is entirely
  yours.

The scope is narrow because the problem it targets — a dependency graph
that's declared once and trusted everywhere — doesn't need to be
anything bigger to solve the drift described above.
