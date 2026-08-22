# Integration testing against real Postgres, Redis, and S3 — in 5 minutes, same commands in CI

By the end of this post you'll have a real Postgres, a real Redis, and a
real S3 bucket (via LocalStack) running on your laptop, health-checked, and
a test suite passing against all three — using one of the actual reference
examples in [`morarix-examples`](https://github.com/morarixhq/morarix-examples),
not a toy snippet built for this post. Here's the whole interaction, before
any explanation:

```
morarix up
pytest
morarix down
```

Three commands. No compose file to write, no mocks to maintain, no
separate CI script. Everything below is showing you those three commands
actually run, with real output.

## The example: python-fastapi

This post walks through
[`python-fastapi`](https://github.com/morarixhq/morarix-examples/tree/main/python-fastapi)
— a FastAPI service backed by Postgres, Redis, and S3. It's one of four
richer, three-dependency reference examples (the others are
`java-spring-boot`, `dotnet-minimal-api`, and `node-typescript`), chosen
here because it demonstrates real range — a database, a cache, and object
storage — rather than "here's a database" in isolation. Everything in this
post uses this one example start to finish.

The service is a small widget API:

- `POST /widgets` — insert a row, invalidate the cached list.
- `GET /widgets` — cache-aside through Redis: hit returns the cached JSON
  with no Postgres round-trip; miss queries Postgres and populates the
  cache with a 30-second TTL.
- `PUT /widgets/{id}/spec` — upload a text spec to S3, `404` if the widget
  doesn't exist in Postgres.
- `GET /widgets/{id}/spec` — fetch that spec back, `404` if it doesn't
  exist.

Nothing exotic — which is the point. This is the shape of dependency most
services actually have.

## The graph

Here's the whole of `morarix.graph.json` for this example:

```json
{
  "schema_version": "1.0",
  "service": {
    "name": "python-fastapi",
    "repo_path": ".",
    "language": "python",
    "framework": "fastapi"
  },
  "dependencies": [
    {
      "id": "db",
      "type": "postgres",
      "fidelity": "real",
      "source": "manual",
      "required_env_vars": ["MORARIX_DB_CONNECTION_STRING"],
      "optional": false,
      "image": "postgres",
      "version": "16"
    },
    {
      "id": "cache",
      "type": "redis",
      "fidelity": "real",
      "source": "manual",
      "required_env_vars": ["MORARIX_CACHE_CONNECTION_STRING"],
      "optional": false,
      "image": "redis",
      "version": "7"
    },
    {
      "id": "assets",
      "type": "s3",
      "fidelity": "real",
      "source": "manual",
      "required_env_vars": ["MORARIX_ASSETS_HOST", "MORARIX_ASSETS_PORT"],
      "optional": false,
      "resource": { "bucket_name": "widget-assets" }
    }
  ],
  "metadata": {
    "authored_at": "2026-08-05T07:05:06.0000000Z",
    "morarix_version": "0.0.1-dev"
  }
}
```

Three things worth calling out:

- **`fidelity: "real"`** on every dependency means Morarix actually
  provisions a container and health-checks it — not "container started,"
  but "Postgres will accept a connection," "Redis responds to `PING`,"
  "the LocalStack S3 endpoint is reachable." `fidelity` can also be
  `stub` (nothing provisioned, your own mocks handle it) or `staging`
  (points at an already-running shared endpoint) — this example just
  doesn't need either.
- **`resource.bucket_name`** on the `assets` dependency means Morarix
  creates the `widget-assets` bucket itself once LocalStack is healthy —
  the app never issues a `CreateBucket` call.
- **This file is hand-authored and git-committed**, not generated magic.
  The `db` entry here started as what a plain `morarix init` scaffolds
  from a from-scratch service; `cache` and `assets` were added by hand
  afterward, in the same shape. There's no inference step anywhere — see
  [why Morarix exists](../concepts/why-morarix.md) for why that's
  deliberate.

## `morarix up`, with real output

This is the actual output from running `morarix up` against this example
— nothing hand-written:

```
$ morarix up
morarix: loaded 3 dependencies for "python-fastapi"
  - db                   postgres       fidelity=real
  - cache                redis          fidelity=real
  - assets               s3             fidelity=real

morarix: environment ready (run_id=morarix-3f16480cfbc9-1786016512, mode=isolated) — morarix.connections.json and .env.morarix written
```

Behind that one "environment ready" line: three containers came up, and
each one was polled until it actually reported healthy — a real accepted
Postgres connection, a real Redis `PING`, a real reachable LocalStack S3
endpoint — not just "the container process started." Only once all three
passed does Morarix write `.env.morarix` and `morarix.connections.json`.
If any dependency doesn't come up healthy within its timeout, nothing gets
written, the containers are left running for inspection (`morarix logs`,
`morarix explain`), and you get a clear error instead of a test suite that
fails somewhere downstream with a confusing connection error.

## Run the existing test suite, unmodified

No test code was written for this post. No mocks. There's no compose file
anywhere in this example repo, either — Morarix is the only thing
provisioning anything. Here's the actual `pytest` run against the
containers `morarix up` just provisioned:

```
$ pytest -v
============================= test session starts =============================
platform win32 -- Python 3.13.2, pytest-9.1.1, pluggy-1.6.0
collecting ... collected 9 items

tests/test_api.py::test_health_reports_ok PASSED                         [ 11%]
tests/test_api.py::test_create_widget_with_no_name_is_rejected PASSED    [ 22%]
tests/test_api.py::test_create_widget_then_list_returns_it PASSED        [ 33%]
tests/test_api.py::test_get_widgets_is_cached_across_consecutive_calls PASSED [ 44%]
tests/test_api.py::test_spec_upload_and_download_round_trips_through_the_api PASSED [ 55%]
tests/test_api.py::test_spec_endpoints_404_for_unknown_widget PASSED     [ 66%]
tests/test_data_access.py::test_insert_and_list_widgets_round_trips_through_postgres PASSED [ 77%]
tests/test_data_access.py::test_cache_set_and_get_round_trips_through_redis PASSED [ 88%]
tests/test_data_access.py::test_spec_upload_and_download_round_trips_through_s3 PASSED [100%]

======================= 9 passed, 3 warnings in 24.13s ========================
```

(Run here on Windows — same commands, same result on Linux or macOS; the
warnings are unrelated library deprecation notices, not anything
Morarix-related.)

That split between `test_api.py` and `test_data_access.py` isn't
accidental — it's two different layers of the same test suite:

- **`test_data_access.py`** talks to the `psycopg` connection, the `redis`
  client, and the `boto3` S3 client *directly* — insert-then-list against
  Postgres, a set/get round-trip against Redis, a put/get round-trip
  against S3. It never invokes a single FastAPI route. It proves the
  dependencies themselves work as expected.
- **`test_api.py`** talks to the app's real HTTP routes through FastAPI's
  `TestClient` — real JSON bodies, real `PUT` uploads — including
  asserting that two consecutive `GET /widgets` calls return an identical
  body (the cache-hit path) and that an uploaded spec is retrievable
  through the API afterward. This is the layer that actually proves
  routing, serialization, and the cache/S3 wiring in `app/main.py` are
  correct end to end — exactly what a mocked unit test can't catch,
  because the mock hides the integration bug this layer exists to find.

The full breakdown of this split — when you need one layer versus both —
is in the
[usage guide's testing philosophy section](../docs/usage-guide.md#3-testing-philosophy--the-three-layers).

## The same thing in CI

This is the direct payoff of "same commands locally and in CI" — not a
parallel CI-specific setup that can quietly drift from what a developer
runs on their laptop, but the literal same three commands:

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"

      - name: Install morarix
        run: go install github.com/morarixhq/morarix/cmd/morarix@latest

      - name: Install dependencies
        run: pip install -r requirements.txt

      - run: morarix up
      - run: pytest
      - run: morarix down
```

`ubuntu-latest` already has Docker available, which is the only real
prerequisite. There's no separate "CI mode" of the graph, no second list
of dependencies to keep in sync with what's declared locally — CI reads
the same `morarix.graph.json` a laptop does, so there's nothing to drift.

## Valid data, not guessed data

This example's Postgres dependency doesn't declare `seed_files` — the app
creates its own `widgets` table on startup instead, the same pattern
`go-postgres` uses. When a dependency *does* need known starting data,
`seed_files` handles it — an ordered list of `.sql` files, applied right
after that dependency reports healthy and before `.env.morarix` is
written. [`node-postgres`](https://github.com/morarixhq/morarix-examples/tree/main/node-postgres)
uses exactly that:

```json
"seed_files": ["seed/schema.sql", "seed/seed_data.sql"]
```

Since every `real` container is freshly created on each `morarix up`,
there's no migration-tracking or idempotency concern — seeding always
starts from an empty database, every run, on every machine, in CI or
local. If you already have your own migration tool (Flyway, Prisma,
golang-migrate), you just don't declare `seed_files` and keep using it —
the two aren't meant to be layered.

## Try it yourself

The full source for this example — app code, tests, the graph, all of
it — is at
[`morarix-examples/python-fastapi`](https://github.com/morarixhq/morarix-examples/tree/main/python-fastapi).
The [usage guide](../docs/usage-guide.md) covers onboarding a service that
doesn't have a graph yet (`morarix init` from scratch, or
`morarix import compose` if you already have a `docker-compose.yml`), plus
the full per-language command reference. And if you're working in Claude
Code, the `morarix-test-skeleton` skill can scaffold the `.env.morarix`
bridge code and a wired, runnable test file for a service that already has
a graph — plumbing only, it doesn't write your test assertions for you,
but it closes the gap between a healthy `morarix up` and your first
running test.
