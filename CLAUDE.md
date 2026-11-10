# CLAUDE.md

Guidelines for Claude Code and other AI assistants working in this repository.
They describe what the project is, how it is built, and the rules every change must follow.
Human contributors follow the same rules.

## Project

`biobank-api` is a REST API for managing biological samples in a biobank: registering
samples, querying and updating them, bulk ingestion from CSV files with row-level
validation, and archiving the original files in Amazon S3.

**Stack**: Python 3.12 · FastAPI · Pydantic v2 · SQLModel · PostgreSQL 16 · pytest ·
ruff · Docker Compose · boto3 (AWS S3).

## Architecture

```
app/
  main.py        # application factory, router registration, lifespan
  config.py      # typed settings loaded from environment variables (pydantic-settings)
  db.py          # engine and session factory
  models.py      # SampleBase, Sample (table), SampleCreate, SampleUpdate, SamplePublic
  deps.py        # shared dependencies (SessionDep, settings, current client)
  security.py    # authentication
  services/      # business logic that does not belong in a route (CSV ingestion, S3)
  routers/       # one module per resource; thin handlers that delegate to services
tests/           # pytest; mirrors the app/ structure
```

**Principles**

- Routers stay thin: parse input, call a service or the database, return a response model.
- One model per purpose: a table model, input models (`Create`, `Update`) and a public
  output model. Table models are never returned directly when they hold internal fields.
- Configuration is injected, never hard-coded: no URLs, bucket names or credentials in code.
- External services (database, S3) are accessed through dependencies so they can be
  overridden in tests.

## Security

These rules are mandatory. A change that breaks one of them is not mergeable.

**Secrets and configuration**

- Never commit secrets: credentials, tokens, keys or a real `.env` file.
- Every setting comes from environment variables. `.env.example` documents each one
  with a placeholder value and must be updated whenever a setting is added.
- Never read, print or copy the contents of `.env`, `~/.aws/` or any credential file
  into code, logs, tests, commit messages or chat output.

**Data protection**

- The project handles health-related data, a special category under the GDPR.
  Only synthetic data is allowed in this repository: fixtures, examples, seeds and test
  files.
- The data model stores pseudonymous identifiers only (e.g. `P0001`). No names, dates of
  birth, national ID numbers, contact details or other direct identifiers.
- Logs must not contain request bodies or sample-level data.

**Application security**

- All input is validated with Pydantic models; constraints (lengths, ranges, enums) live
  in the model, not in ad-hoc checks.
- No SQL built with string formatting or concatenation; use SQLModel queries or bound
  parameters.
- Every endpoint requires authentication except `GET /health` and the API docs.
- Error responses never expose stack traces, SQL statements, file paths or internal IDs
  that are not part of the public contract.
- Uploaded files are size-limited, type-checked and parsed defensively; a malformed row
  is reported, never silently skipped.

**AWS**

- Least privilege: the application's IAM permissions cover only the S3 actions it uses
  on its own bucket. Root credentials are never used.
- In deployed environments the application uses an IAM role, never long-lived access keys.
- Buckets are private with *Block Public Access* enabled; files are shared only through
  short-lived presigned URLs.
- Tests never call real AWS services; S3 is mocked (`moto`) or overridden.

**Dependencies**

- Dependencies are declared and pinned in `pyproject.toml`. A new dependency needs a
  clear reason and should be the smallest well-maintained option.
- Dependabot alerts and secret scanning stay enabled.

## Coding standards

- Type hints on every function signature; `ruff` for linting and formatting
  (`ruff check .` and `ruff format .` must be clean).
- Small, single-purpose functions. Docstrings on public functions and on every endpoint
  (they are published in the OpenAPI documentation).
- Explicit over clever: readable code is preferred to compact code.
- No commented-out code, no unused imports, no `print` debugging in committed code;
  use the `logging` module.

## API conventions

- Resources are plural nouns (`/samples`); JSON request and response bodies.
- Status codes: `200` read/update, `201` create, `204` delete, `401` unauthenticated,
  `404` not found, `409` conflict, `422` validation error.
- Collections are paginated with `offset` and `limit` (default 20, maximum 100).
- Partial updates use `PATCH` and only modify the fields sent by the client.
- Breaking changes to the public contract are not allowed without updating the README
  and the version.

## Testing

- Every endpoint has tests for the success path and for its error paths (validation,
  not found, unauthenticated).
- Tests use an in-memory SQLite database injected through dependency overrides, so the
  suite runs without Docker or network access.
- A change is done only when `pytest` and `ruff` pass locally and in CI.

## Workflow

- `main` is protected: all changes go through a pull request and are squash-merged.
- Branches: `feat/…`, `fix/…`, `refactor/…`, `test/…`, `docs/…`, `chore/…`.
- Commits follow Conventional Commits (`feat: add PATCH /samples/{id}`).
- One logical change per pull request. The README is updated in the same pull request
  when behaviour, configuration or commands change.

**Rules for AI assistants**

- Before changing code, state the plan: files affected, expected behaviour and tests.
- Keep changes scoped to the task; do not refactor unrelated code in the same change.
- Do not add dependencies, change the architecture or the public API contract without
  explicit approval.
- Never commit, push, open or merge pull requests without explicit approval.
- Destructive operations (`git push --force`, history rewrites, `rm -rf`, dropping
  databases, deleting cloud resources) always require explicit confirmation.

## Commands

```bash
fastapi dev app/main.py            # run locally with auto-reload (docs at /docs)
pytest                             # run the test suite
ruff check . && ruff format .      # lint and format
docker compose up --build          # API + PostgreSQL
```

## Language

Code, comments, commit messages, pull requests and documentation are written in English.
