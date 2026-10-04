# Tidewater: Argyle Paystub Sync, Take-Home Assignment

## Your Task

Build a service that receives Argyle paystub webhooks, validates them, and syncs the
account's paystubs from the Argyle API into a database. Use any language and framework you
like.

**Time estimate:** 4 hours

---

## What You'll Build

A service in a Docker image that meets this contract.

| Endpoint | What it does |
|---|---|
| `POST /webhooks/argyle` | Receives Argyle webhook deliveries |
| `POST /internal/sync` | Starts a sync for `{"userId", "incomeId", "argyleUserId", "argyleAccountId"}` and answers `202` with `{"run_id": "<id>"}` right away, where `<id>` is the `id` of the `sync_runs` row for that sync |
| `GET /healthz` | Answers `200` once the service can take traffic |

Your service gets these environment variables:

| Variable | What it is |
|---|---|
| `PORT` | The port to listen on |
| `DATABASE_PATH` | The SQLite database to use (see [Database](#database)) |
| `ARGYLE_BASE_URL` | The Argyle API |
| `ARGYLE_API_ID`, `ARGYLE_API_SECRET` | Basic-auth credentials for the Argyle API |
| `ARGYLE_WEBHOOK_SECRET` | The secret webhook deliveries are signed with |

The API is documented in [`docs/api/`](docs/api/): [overview](docs/api/overview.md),
[paystubs](docs/api/paystubs.md) and [webhooks](docs/api/webhooks.md).

---

## Acceptance Criteria

### Webhook handler (`POST /webhooks/argyle`)
- [ ] Verify the `X-Argyle-Signature` header (HMAC-SHA512 of the raw body): return 401 if invalid
- [ ] Validate the payload: return 400 if invalid
- [ ] Log every delivery to `webhook_events`, failures included
- [ ] For the paystub events the provider sends: find the income by `external_account_id`, write a `sync_runs` row with status `pending` before you return 200, and run the sync
- [ ] Return 200 on success

### Sync
- [ ] Fetch the account's paystubs from the Argyle API, handling pagination
- [ ] Upsert paystubs: insert new ones, update existing ones (matched by `external_id`)
- [ ] Move the `sync_runs` row from `pending` through `running` to `succeeded` or `failed`

### Service
- [ ] `POST /internal/sync` and `GET /healthz` as in the table above
- [ ] `POST /internal/sync` writes its `sync_runs` row (status `pending`) before it answers
  `202`, and returns that row's `id` as `run_id`
- [ ] A `Dockerfile` whose final stage builds `FROM ghcr.io/latentsp-hiring/tidewater-base:2`

> **The acceptance criteria are the floor, not the ceiling.** They describe what the
> integration does when everything goes right. See "Don't assume the other side behaves"
> below.

---

## Don't assume the other side behaves

You are integrating with a system you do not control, over a network you do not control.
The mock server is written to behave like a real third-party provider rather than a
well-mannered test fixture: it does not always do what its documentation implies, and it
does not always do the same thing twice.

We are not going to tell you how it misbehaves — working that out is part of the exercise.
Run it, send it traffic, watch what actually arrives at your endpoint and what the API
actually returns, and decide for yourself what your code needs to survive.

Then say in your write-up what you found and what you chose to handle. **We would much
rather read "I noticed X and deliberately didn't handle it because Y" than see it silently
unhandled.** Deciding what to skip in 4 hours is part of the answer, and we grade the
reasoning, not just the code.

---

## The service contract

Your `Dockerfile` can use any stack in earlier stages. Its **final stage** must build
`FROM ghcr.io/latentsp-hiring/tidewater-base:2` (its source is in
[`images/base/`](images/base/)) and put two executables in place:

- **`/app/start`** starts your service in the foreground.
- **`/app/migrate`** prepares the database. The base image already ships a default that
  applies `/app/migrations/*.sql` in filename order and records each file it applied in a
  `_migrations` table, so most people never touch it. You may
  replace it with your own tool (for example a wrapper around `bin/rails db:migrate` or
  `mix ecto.migrate`); a replaced `/app/migrate` is yours to maintain.

The default migrate reads your migrations from `/app/migrations` inside the image. If your
final stage does not `COPY . /app` (a multi-stage build, for example), it must
`COPY migrations /app/migrations`. The default runs each file in its own transaction. So a
migration must not contain `BEGIN;`, `COMMIT;` or `ROLLBACK` statements, or sqlite3
dot-commands such as `.read`. A `CREATE TRIGGER` body may still use `BEGIN ... END`.

The grader runs `/app/migrate` once, on a fresh database, before it runs `/app/start`. It
executes `/app/migrate` directly and sets only `DATABASE_PATH`, on top of your image's
`ENV`. The default skips files it has already applied, so running it again (from
`/app/start`, or on your `data/dev.db`) is safe. A replacement must be safe to run again
too if you call it from `/app/start`.

If `/app/start` or `/app/migrate` is a script, it must begin with a `#!` line and use LF
line endings. A native executable works as either one. The repo's `.gitattributes` keeps
`start`, `migrate` and `*.sh` LF on Windows checkouts too.

A minimal Dockerfile for a Python service:

```dockerfile
FROM ghcr.io/latentsp-hiring/tidewater-base:2
COPY . /app
RUN chmod +x /app/start
```

You do not need Docker on your own machine: run your service directly while you work,
and use the practice validator below to check the image builds.

### How the grader runs your image

- It builds and runs your image on linux/amd64.
- Your image must build within 10 minutes.
- The running image has no network access. Install everything at build time.
- `GET /healthz` must answer `200` within 60 seconds of `/app/start` starting.
- `POST /internal/sync` must answer within 10 seconds.
- A sync must reach `succeeded` or `failed` within 2 minutes.
- `/app/migrate` must finish within 2 minutes.

---

## Database

The grader creates the SQLite database at `$DATABASE_PATH` from
[`contract/schema.sql`](contract/schema.sql) and [`contract/seed.sql`](contract/seed.sql),
then runs your `/app/migrate`. Your service must not create or change tables at runtime:
put schema changes in `migrations/NNNN_description.sql`.

You may **extend** the schema freely: new tables, columns, indexes and constraints. You
must **never remove or retype** these columns, because the grader reads them:

- `users`: `id`
- `incomes`: `id`, `external_account_id`
- `paystubs`: `external_id`, `user_id`, `gross_pay` (REAL)
- `sync_runs`: `id`, `user_id`, `status`, `created_at`, `finished_at`, `error`
- `webhook_events`: `failed_at`, `error`
- the seeded rows in `users` and `incomes`

Every column in this list is TEXT except `gross_pay`. A `user_id` column holds a
`users.id`, such as `test-user-1`. `paystubs.external_id` holds the Argyle paystub's `id`.

Build your local database the same way the grader does (needs the `sqlite3` CLI):

```bash
sh scripts/init-db         # macOS / Linux
.\scripts\init-db.ps1      # Windows (PowerShell)
```

This writes `data/dev.db`. **Commit `data/dev.db` with your work**, including the traffic
your service recorded while you developed against the mock. Stop your service before you
commit it. SQLite can keep recent rows in a separate write-ahead log
(`data/dev.db-wal`) until the last connection closes, and git ignores that file.

Run `init-db` once. If `data/dev.db` already exists, it refuses to run, because rebuilding
deletes the traffic you recorded. To rebuild from scratch anyway, run
`sh scripts/init-db --force`. When you add a migration later, run the default migrate
script on your existing `data/dev.db`. It is the one the grader runs as `/app/migrate`
unless you replace it, and it applies only the files it has not applied yet:

```bash
DATABASE_PATH=data/dev.db MIGRATIONS_DIR=migrations sh images/base/migrate
```

On Windows, run it from Git Bash.

If you replaced `/app/migrate`, run your own script in that command instead, and set
`MIGRATE` to its path before you run `init-db`. `init-db` runs it with `sh`, with
`DATABASE_PATH` and `MIGRATIONS_DIR` set. The grader does not use `sh` or set
`MIGRATIONS_DIR`: it executes `/app/migrate` directly, with only `DATABASE_PATH` set.

---

## Setup

### Requirements
- The toolchain for the stack you choose
- The `sqlite3` CLI
- macOS, Linux, or Windows

### Run the mock and your service

```bash
./scripts/run-mock-server                     # Terminal 1: the Argyle mock, on :8080
sh scripts/init-db                            # once (see Database above)
PORT=3000 DATABASE_PATH=data/dev.db \
ARGYLE_BASE_URL=http://localhost:8080 ARGYLE_API_ID=mock-id ARGYLE_API_SECRET=mock-secret \
ARGYLE_WEBHOOK_SECRET=your-webhook-secret  <start your service>   # Terminal 2
```

On Windows (PowerShell):

```powershell
.\bin\argyle-mock-windows-amd64.exe              # Terminal 1 (argyle-mock-windows-arm64.exe on ARM)
.\scripts\init-db.ps1                            # once (see Database above)
$env:PORT = "3000"; $env:DATABASE_PATH = "data/dev.db"
$env:ARGYLE_BASE_URL = "http://localhost:8080"; $env:ARGYLE_API_ID = "mock-id"; $env:ARGYLE_API_SECRET = "mock-secret"
$env:ARGYLE_WEBHOOK_SECRET = "your-webhook-secret"; <start your service>   # Terminal 2
```

---

## Testing Your Implementation

### 1. Register your webhook

```bash
curl -X POST http://localhost:8080/webhooks \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test",
    "url": "http://localhost:3000/webhooks/argyle",
    "secret": "your-webhook-secret",
    "events": ["paystubs.added", "paystubs.updated", "paystubs.partially_synced", "paystubs.fully_synced"]
  }'
```

> The `secret` must match your `ARGYLE_WEBHOOK_SECRET`.

### 2. Trigger a test sync

```bash
curl -X POST http://localhost:8080/simulate/connect-seeded
```

This uses the pre-seeded account ID (`019b41d0-7a84-72db-beab-4f62f8e86ce4`) that matches a database income record.

### 3. Verify

Look at `data/dev.db` (`sqlite3 data/dev.db 'select count(*) from paystubs'`).

---

## Practice validator

```bash
./run.sh validate --preflight      # builds your image and checks it starts and is wired up
./run.sh validate                  # a scored attempt: PASS n / m
```

On Windows, use `.\run.ps1 validate --preflight` and `.\run.ps1 validate`. `run.ps1`
picks the CLI build for your PowerShell's architecture, and there is no Windows ARM build
of the CLI. If `run.ps1` reports `Binary not found` on Windows on ARM, run the amd64 build
directly, for example `.\bin\latent-cli-windows-amd64.exe validate --preflight`. Windows
11 on ARM runs it under x64 emulation. Use the same binary for `publish`.

`--preflight` tells you exactly what is wrong when your image does not build, start, or
reach its database. It does not use an attempt. You can run it 20 times a day; the count
resets at 00:00 UTC. A scored attempt reports only how many checks passed, never which.
You have **three**, so save them for when you think you are done. If a scored attempt
fails its preflight, it shows the preflight errors and uses no attempt.

### What validate sends

Both commands send your files that git does not ignore, committed or not, except:

- `.env` and `.env.*` files
- dependency and build folders: `node_modules`, `.venv`, `venv`, `__pycache__`, `.next`,
  `.turbo`, `dist`, `build`
- the `argyle-mock-*` and `latent-cli-*` binaries in `bin/`
- `data/dev.db`

So your Dockerfile must build anything those folders would hold. The compressed archive
must be at most 8 MiB.

### Attempts

Attempts are counted per identity: the part of your email before the `@`, in lower case.
Characters other than letters, digits, `.`, `_` and `-` become `-`. The CLI reads your
email from `git config user.email` unless you pass `--email you@example.com`.

A scored attempt can take up to about 40 minutes. Leave it running. If the CLI stops
waiting, run the same command again without changing your files: it shows that run's
result and uses no attempt. Validating files that already have a scored result shows that
result again and uses no attempt. Only a completed scored attempt is reused this way. A
preflight always runs again, and every preflight counts toward its daily limit.

---

## Mock Server API

| Endpoint | Description |
|----------|-------------|
| `POST /webhooks` | Register webhook URL |
| `POST /simulate/connect-seeded` | Trigger webhooks for seeded account |
| `GET /paystubs?account={id}&limit={n}` | List paystubs (paginated via `next`/`previous` cursor URLs) |

See [`docs/api/`](docs/api/) for the full reference.

The mock injects failures. `--chaos-level` and `--api-chaos-level` (0–100, default 25)
set how often. Set both to `0` for a quiet baseline.

---

## Constraints & notes

- **AI / coding agents are welcome for the code.** Use Copilot, Cursor, Claude, ChatGPT,
  or whatever you like — we expect you to. How you use them, and where you apply your own
  judgment on top, is part of what we're interested in.
- **`WRITEUP.md` is yours alone.** Do not use an AI to write it. It is short, and it is
  the part of the submission we read most carefully — we want your reasoning in your
  words, not a model's summary of your code. Non-native English, typos and rough edges are
  completely fine and are never held against you.
- Commit as you go. We read the git history, and an honest history with dead ends in it
  reads better than one tidy commit.

---

## Write-up

Add a `WRITEUP.md` at the repo root. Keep it short — half a page to a page is plenty.
Cover:

1. **What you found.** What does the other side actually do that the spec doesn't mention?
2. **What you handled, and how.** The design decisions you'd defend in review.
3. **What you deliberately didn't handle,** and why — time, risk, or judgment.
4. **What you'd do next** with another day.

---

## Required artifacts

Commit these with your work:

- your service's source and a `Dockerfile` at the repo root
- `data/dev.db`, the database your service wrote while you developed
- `migrations/`, if you changed the schema
- `WRITEUP.md`
- your git history as it happened: please do not squash it

---

## Repository layout

```
docs/api/                 # the Argyle API reference
contract/schema.sql       # the database the grader creates
contract/seed.sql         # its fixture rows
migrations/               # your schema changes, applied by /app/migrate
data/dev.db               # yours: commit it
images/base/              # source of the base image your Dockerfile builds FROM
scripts/run-mock-server   # runs the Argyle mock
scripts/init-db           # builds data/dev.db the way the grader builds its database
scripts/init-db.ps1       # the same, for Windows (PowerShell)
bin/
  argyle-mock-*           # mock Argyle server (do not edit)
  latent-cli-*            # submission CLI (do not edit)
run.sh / run.ps1          # submission wrappers (macOS·Linux / Windows)
Dockerfile                # yours
start                     # yours: starts your service; copied to /app/start
WRITEUP.md                # yours
```

---

## Submission

When you're ready to submit, use the provided CLI to bundle and upload your repository.
From the root of your assignment repo:

```bash
./run.sh publish        # macOS / Linux
.\run.ps1 publish       # Windows (PowerShell)
```

This bundles your git repository — including the `.git` history we review, and respecting
`.gitignore` (so `.env`, `node_modules`, etc. are excluded) — and uploads it to our
evaluation system. Unlike `validate`, it includes `data/dev.db`. It also includes two logs
kept under `~/.latent/`: `validation-log.json`, the CLI's log of your validate runs, and
`argyle-mock-journal.jsonl`, the mock's request log. To check what it would send without
uploading anything, write the bundle outside the repo and list it:
`./run.sh publish --out ../bundle.tgz && tar -tzf ../bundle.tgz`.

### Your AI conversations

If you used AI coding tools, `publish` asks whether to include your conversations with them
for this task (`[Y/n]`). It's optional, and you can submit either way. If it finds none on
your computer, for example because you chatted in a browser, it still asks `[Y/n]` first.
After a yes, it asks for the path to an exported copy (Enter skips). To answer without the
prompt, pass `--transcripts=yes` or `--transcripts=no`. To attach an exported
conversation, pass `--transcript-file path/to/chat.md` (repeat it for several files). An
attached file is included whatever you answer, `--transcripts=no` included.

Before upload, the CLI removes recognized credentials and personal details, such as
emails and phone numbers, from the conversations and the attached exports. To check the
result, write the bundle with `--out` first and read it.

### Providing Your Email

The CLI uses your email address to identify your submission and your validate attempts.
It will:

1. First try to read your email from `git config user.email`
2. If it is not set, prompt you to enter it interactively
3. Alternatively, pass it directly: `./run.sh publish --email you@example.com`

### Confirmation

After the upload completes, you will be asked to reply to the email you received for this
assignment with your name and the repository name. Please send that reply so we can match
your submission to your application.

---

## Documentation

- [`docs/api/`](docs/api/): the reference for the mock you integrate against. Start here.
- [Argyle Paystubs Webhooks](https://docs.argyle.com/api-reference/paystubs-webhooks) and
  [Argyle Paystubs API](https://docs.argyle.com/api-reference/paystubs): the real API it models.

---

## Questions?

If you're blocked on setup issues (not implementation), reach out. Good luck!
