# Morarix Usage Guide

A step-by-step guide for engineers new to Morarix: how to onboard a service,
how to think about testing against it, and the exact commands to run
locally and in CI.

## 1. What Morarix does

Morarix reads a dependency graph you author (`morarix.graph.json`),
provisions your service's dependencies (Postgres, Redis, Kafka,
LocalStack, etc.) as real Docker containers, seeds them if you asked it
to, health-checks them, and writes the connection info to
`.env.morarix` / `morarix.connections.json`.

**Morarix never touches your application.** It doesn't build an image
for it, doesn't containerize it, and doesn't run it. Your app always runs
as a native local process — on your laptop and in CI alike — using
whatever language/runtime tooling you already use. Morarix's job ends the
moment your dependencies are healthy and their connection details are on
disk.

```
                 ┌─────────────────────────┐
                 │  morarix.graph.json      │   you author this
                 └────────────┬────────────┘
                              │ morarix up
                              ▼
                 ┌─────────────────────────┐
                 │  Postgres, Redis, ...    │   real containers
                 │  (docker compose)        │
                 └────────────┬────────────┘
                              │ writes
                              ▼
                 ┌─────────────────────────┐
                 │  .env.morarix            │   credentials, ports
                 └────────────┬────────────┘
                              │ read by
                              ▼
                 ┌─────────────────────────┐
                 │  your app / your tests   │   native process,
                 │  (go run, npm start,     │   never Morarix's
                 │   go test, npm test...)  │   concern
                 └─────────────────────────┘
```

## 2. Onboarding a service — step by step

**Step 1 — get a graph.**

| Situation | Command | Decision point |
|---|---|---|
| No existing `docker-compose.yml` | `morarix init` | Scaffolds a template graph for you to edit by hand |
| Already have a `docker-compose.yml` with your dependencies | `morarix import compose` | One-time, deterministic conversion. Services with no `image:` (i.e. your own app, built from a `Dockerfile`) are skipped automatically |

Either way, the result is the same: a hand-editable, git-committable
`morarix.graph.json` with no secrets in it.

**Step 2 — set `fidelity` per dependency.** This is the main decision
point for cost/speed vs. realism:

| Fidelity | What Morarix does | When to use it |
|---|---|---|
| `real` | Spins up a genuine container, health-checks it | You need the real behavior (SQL semantics, Redis eviction, Kafka offsets...) |
| `stub` | Does nothing — health is `not-applicable`, never blocks readiness | You already mock this dependency in your own tests |
| `staging` | Points at an already-running shared endpoint, does a lightweight reachability check | Dependency is expensive/slow to spin up locally (e.g. a shared LocalStack cluster) and a shared instance already exists |

**Step 3 — (optional) declare `seed_files`** on a `real` Postgres/MySQL
dependency — an ordered list of `.sql` files. Morarix applies them right
after that dependency reports healthy, before writing `.env.morarix`. Skip
this entirely if you already have your own migration tool (Flyway, Prisma,
golang-migrate...) — `seed_files` and your own migration tooling are
mutually exclusive, not layered.

**Step 4 — run it.**

```
morarix up      # provisions dependencies, seeds them, writes .env.morarix
morarix status  # confirm everything's healthy
morarix down    # tear down when you're done
```

If something doesn't come up healthy, `morarix explain` and `morarix
logs` are your first stops before `morarix doctor`.

## 3. Testing philosophy — the three layers

This is the part engineers most often ask about, so it's worth being
explicit. There are three distinct things you can do once `morarix up`
has your dependencies healthy, and they are **complementary, not
alternatives** — a mature test suite uses more than one.

### Layer 1 — dependency-only tests (no app running at all)

You talk to the dependency directly — a DB pool, a Redis client, a Kafka
producer — using the credentials from `.env.morarix`. Your application
code (routes, handlers, `main()`) is never invoked.

- **Proves:** the dependency itself is reachable and behaves as expected;
  your data-access layer's queries are correct.
- **Doesn't prove:** that your app's HTTP routes, request/response DTOs,
  or wiring between layers are correct — because none of that code ran.
- **Example:** `main_test.go` / `widgets.test.js` in the reference
  examples — talk to `*sql.DB` / the `pg` pool directly.

### Layer 2 — app running locally, invoking its own API

Your application is started as a real process (or, for automated tests,
in-process via `httptest.NewServer` in Go or `server.listen(0)` in Node),
and you call it exactly as an external client would — real HTTP requests,
real JSON request/response bodies.

- **Proves:** routing, DTO serialization/deserialization, and the
  wiring between your handler and your real dependency all work
  end-to-end. This is exactly what a mocked unit test *can't* catch,
  because the mock hides the very integration bugs this layer is meant to
  find — and it's what Layer 1 skips, since Layer 1 never goes through
  your HTTP layer at all.
- **Doesn't prove:** anything about containerization, deployment, or
  networking beyond localhost.
- **Example:** `api_test.go` / `api.test.js` in the reference examples —
  `httptest.NewServer(newMux(db))` / `server.listen(0)`, then real
  `fetch`/`http.Get`/`http.Post` calls.
- **Manual equivalent:** `go run .` / `npm start`, then `curl` — same
  layer, just exploratory instead of automated.

### Layer 3 — app running inside a container

Deliberately **not** something Morarix does. If you want to test your
app's own Dockerfile/image (build correctness, entrypoint, base-image
issues), that's a separate concern from dependency provisioning — build
and run your own image yourself, pointed at the same `.env.morarix`
values Morarix already wrote. Most teams only need this for a pre-release
smoke test, not for everyday test runs, precisely because Layers 1 and 2
already run against the real dependency and don't need image-build
overhead to do it.

### Decision point: which layers do I need?

- Writing a new endpoint? Write a Layer 2 test — it's the one that
  actually proves the endpoint works.
- Writing complex query/data-access logic? A Layer 1 test alongside it is
  cheap and pinpoints failures closer to the source.
- Already have unit tests with mocked dependencies? Keep them for fast
  feedback on pure business logic — they are not a substitute for Layer 2,
  since a mock can't catch a wiring or serialization bug by construction.
- Shipping a container image? That's the only case Layer 3 earns its
  keep, and only occasionally (pre-release smoke test), not per-commit.

## 4. Local test-running patterns

**Pattern A — dependency-only (Layer 1), no app process at all:**

```
morarix up
<run your Layer 1 tests — e.g. go test ./... targeting *_test.go that only touch the DB, or npm test targeting DB-only files>
morarix down
```

**Pattern B — manual exploration against a running app (Layer 2, by hand):**

```
morarix up
go run .          # or: npm start / mvn spring-boot:run / dotnet run / python app.py / rails server
curl localhost:3000/widgets
morarix down
```

**Pattern C — automated API-layer tests (Layer 2, in-process):**

```
morarix up
go test ./...     # or: npm test / pytest / mvn test / dotnet test / rspec
morarix down
```

In practice, Patterns A and C run from the **same test command** — `go
test ./...` / `npm test` picks up every test file, so a single invocation
covers both Layer 1 and Layer 2 tests if you've written both (exactly
what the reference examples do: `main_test.go` + `api_test.go`,
`widgets.test.js` + `api.test.js`).

## 5. CI patterns

CI runs the **identical** commands — no separate "CI mode," no
containerized app, because CI runners are already ephemeral and standard
CI actions already provide the right language runtime:

```yaml
# same idea in any CI system
- run: morarix up
- run: go test ./...      # or npm test, pytest, mvn test, dotnet test, rspec
- run: morarix down
```

Because the app is a native process in both environments, there is no
environment drift to design around — the same test files that catch a
wiring bug on your laptop catch it in CI, with the same commands.

## 6. Per-language quick reference

| Language | Run the app | Run tests |
|---|---|---|
| Node.js | `npm start` | `npm test` |
| Go | `go run .` | `go test ./...` |
| Python | `python app.py` (or `uvicorn app:app`) | `pytest` |
| Java | `mvn spring-boot:run` | `mvn test` |
| .NET | `dotnet run` | `dotnet test` |
| Ruby | `rails server` (or `ruby app.rb`) | `rspec` |

All of these read `.env.morarix` (or `morarix.connections.json`) using
whatever env-loading convention is idiomatic for that language —
`dotenv`/`python-dotenv` where the ecosystem has one, or a small custom
loader where it doesn't (Go, Java, and .NET all lack a stdlib/idiomatic
way to load a dynamically-generated env file, so the reference examples
for those three hand-roll one instead of relying on static config files
like `application.properties`/`appsettings`, which don't fit
runtime-generated values well).

## 7. Reference examples

Six working, end-to-end examples live in the `morarix-examples` repo.
Two are minimal, single-dependency examples — one per onboarding path:

- **node-postgres** — onboarding an existing `docker-compose.yml` via
  `morarix import compose`, with `seed_files`-driven schema/data.
- **go-postgres** — onboarding from scratch via `morarix init`, with the
  app bootstrapping its own schema.

The other four are richer, multi-dependency examples (Postgres + Redis +
S3/LocalStack each — cache-aside reads and an S3-backed file upload/
download on top of the same Postgres-backed resource), one per remaining
language in the table above:

- **java-spring-boot**, **dotnet-minimal-api**, **node-typescript**,
  **python-fastapi**.

All six demonstrate Layer 1 and Layer 2 tests side by side, and are the
best starting point if you want to see this guide's commands in a real
repo rather than in the abstract.

## 8. AWS / LocalStack services

Morarix supports eight AWS services against a `fidelity: real` dependency,
all backed by [LocalStack](https://localstack.cloud): S3, SQS, SNS,
DynamoDB, Kinesis, Lambda, Secrets Manager, and SSM Parameter Store.

Every LocalStack-backed dependency in a graph shares **one** container
(`SERVICES=s3,sqs,...`) — the idiomatic way to run LocalStack — rather
than one container per service. `morarix init` offers all eight as
dependency choices alongside Postgres/Redis/Kafka/etc.

Morarix pins this container to `localstack/localstack:4.0.3`, not
`:latest`. Since LocalStack 2026.3.0, the `:latest` tag merged the
community and pro images into one, and that image refuses to start at all
without a `LOCALSTACK_AUTH_TOKEN` — even for the free-tier services this
feature only ever uses (S3, SQS, SNS, DynamoDB, Kinesis, Lambda, Secrets
Manager, SSM). `4.0.3` predates that merge and runs with no token and no
signup. If you override `image`/`version` on a LocalStack-backed
dependency yourself, keep this in mind — an unpinned `:latest` will fail
immediately with a license error, not a flaky timeout.

### Auto-creating the actual resource

Declaring the dependency only gets you the container with that service
enabled. To have `morarix up` also create the bucket/queue/table/etc. (or
register a Lambda function) automatically — instead of you creating it
yourself with a script or the AWS CLI — add a `resource` block:

```json
{
  "id": "uploads",
  "type": "s3",
  "fidelity": "real",
  "resource": { "bucket_name": "uploads" }
},
{
  "id": "orders",
  "type": "dynamodb",
  "fidelity": "real",
  "resource": { "table_name": "orders", "partition_key": "order_id" }
},
{
  "id": "notify",
  "type": "lambda",
  "fidelity": "real",
  "resource": {
    "runtime": "nodejs20.x",
    "handler": "index.handler",
    "zip_file": "dist/notify.zip"
  }
}
```

`resource` is optional per dependency — omit it and Morarix only enables
the service in the shared container, same as before this existed. The
fields that apply depend on `type`:

| Type | `resource` fields |
|---|---|
| `s3` | `bucket_name` |
| `sqs` | `queue_name` |
| `sns` | `topic_name` |
| `dynamodb` | `table_name`, `partition_key`, `sort_key` (optional) — always `PAY_PER_REQUEST`, no throughput to configure |
| `kinesis` | `stream_name`, `shard_count` (defaults to 1) |
| `ssm-parameter-store` | `parameter_name`, `parameter_value_env` |
| `secrets-manager` | `secret_name`, `secret_value_env` |
| `lambda` | `runtime`, `handler`, `zip_file` (required together), `environment` (optional) |

Resource creation runs once the dependency's container reports healthy,
right before `.env.morarix`/`morarix.connections.json` are written —
the same lifecycle slot `seed_files` already occupies. A creation failure
behaves exactly like a seeding or health-check failure: containers stay
up, the error is printed, and `morarix down` cleans up.

### Lambda: zip packages only

Morarix registers a **zip-based** Lambda function — you build the zip
with your own tooling (`npm run build && zip`, `sam build`, etc.);
Morarix never builds it, only uploads it (consistent with "no image
builds, ever"). **Image-based (`PackageType: Image`) functions are not
supported** — they're heavier to build and push even for local iteration,
and zip covers the common case. This may be revisited later if there's
real demand.

**Requires Docker-in-Docker.** LocalStack's Lambda executor runs each
registered function in a sibling container, so whenever a graph declares
a `lambda` dependency, Morarix mounts the host's Docker socket
(`/var/run/docker.sock`) into the shared LocalStack container. Without
it, `CreateFunction` still succeeds, but the function itself sits in
`State: Failed` ("Docker not available") indefinitely — this is handled
for you automatically, but worth knowing if you're inspecting the
generated compose file. The very first Lambda ever created on a machine
also has to pull the matching `public.ecr.aws/lambda/<runtime>` image,
which can take a few minutes beyond the normal cold-start window — a
one-time cost; it's cached for every run after.

### Secrets and parameter values are never in `morarix.graph.json`

Notice `secret_value_env`/`parameter_value_env` name an **environment
variable**, not a literal value. `morarix.graph.json` is git-committable
by design and must never contain a secret or live value — so Morarix
reads the actual secret string from that env var on your machine at
`morarix up` time, the same trust boundary as any other local credential.
Set it in your shell (or a local, gitignored `.env` your shell loader
picks up) before running `morarix up`; don't put the value in the graph.
