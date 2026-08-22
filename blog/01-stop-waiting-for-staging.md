# Stop waiting for staging

It's 2pm. You need to ship a fix before end of day. You run the integration
suite before opening a PR, and it fails — not because your code is wrong,
but because staging is down. Someone on another team pushed a bad migration
an hour ago. You didn't do anything. You're not even sure yet if it's your
bug or theirs. You're blocked on a Slack thread now, not on your own work,
and the fix that was going to take twenty minutes is going to take the rest
of the afternoon.

If you've worked anywhere with a shared test environment, you've lived some
version of this. It's not a rare failure mode — it's the default failure
mode, because a shared environment has exactly one copy of "everything
working," and everyone using it is one bad deploy away from not having it.

## The two options, and why both are bad

Ask most teams how they test against their dependencies and you'll get one
of two answers.

**Mock everything.** Fast, deterministic, runs anywhere. Also proves
nothing about how your code behaves against the real thing. A mocked Redis
client doesn't tell you your eviction policy assumption was wrong. A mocked
Postgres doesn't catch the query that works fine against your mock's
in-memory map and then times out against a real index. The mock passes
right up until the moment the real dependency doesn't behave like the mock
assumed it would — and that moment is always in production, never in CI.

**Share a staging environment.** Real behavior, real bugs caught early.
Also slow to provision, expensive to keep running, and a single point of
failure shared across every team that depends on it. One bad deploy takes
it down for everyone queued behind it — and until someone notices, fixes
it, and redeploys, your "fast feedback loop" is however long that takes.

Neither of these is a bad decision by the people who made it. They're the
two options that exist once you've decided dependencies have to be either
faked or shared. The actual cost isn't just the occasional outage — it's
the waiting: waiting for staging to come back, waiting for a cloud deploy
cycle just to find out if an integration works at all, waiting behind
other teams' incidents that have nothing to do with your change.

## The reframe

You don't need a *shared* environment to test against something *real*.
You need a disposable, isolated, prod-like environment — and there's no
reason it can't live on your own machine, or in your own CI job, instead
of on a server everyone else is also depending on. Real dependency
behavior and private-per-run aren't in tension; they only look that way
because "real" has historically meant "provisioned once, centrally, and
shared."

That's the actual shift: keep the realism, drop the sharing.

## What Morarix actually is

In one sentence: you declare your service's dependency graph once, in a
small git-committed file, and `morarix up` gives you real Postgres, Redis,
Kafka, or S3 (via LocalStack) — health-checked, seeded with valid data if
you want it — on `localhost`, in about as long as it takes those
containers to become healthy. Same command locally and in CI. No shared
server to wait for, and nobody else's incident can block you, because
there's no "everyone else" using your instance.

One thing worth being precise about, because it's the difference between
a claim that holds up and one that gets picked apart in the first comment:
this is about testing *your service* against *its own* real dependencies.
It is not a replacement for a multi-service, dozens-of-teams staging
environment, and it doesn't try to be. If your integration problem is "does
service A correctly call service B correctly calling service C," that's a
different, harder problem. If your integration problem is "does my code
actually work against a real Postgres, not my mental model of one," that's
this one.

## The payoff

Every minute spent waiting for a shared environment to come back, or
writing yet another mock that will quietly drift from how the real
dependency actually behaves, is a minute not spent shipping. That's the
actual pitch — not "a new tool," but deleting a whole category of waiting,
and a whole category of "who broke staging today," from a team's day. A
test run that used to depend on a Slack thread now depends on nothing but
your own machine and a Docker daemon.

If you want the fuller argument for why the drift between "what's declared"
and "what's actually running" is the real underlying problem — not just
staging outages, but schema drift, CI configs that quietly diverge from
local, onboarding that takes a week because nobody wrote the setup down —
that's in [why Morarix exists](../concepts/why-morarix.md).

If you want to see it work — actual commands, actual output, a real
Postgres/Redis/S3 stack coming up and a test suite passing against it, in
about five minutes — that's the next post:
[Integration testing against real Postgres, Redis, and S3 — in 5 minutes, same commands in CI](02-integration-testing-in-5-minutes.md).

Or skip straight to it: [github.com/morarixhq/morarix](https://github.com/morarixhq/morarix).
