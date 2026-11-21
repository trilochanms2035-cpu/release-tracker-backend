# Release Tracker API

A REST API for tracking projects and the tasks inside them, built with FastAPI, SQLModel, and PostgreSQL. It has JWT login, task filtering, Alembic migrations, tests, and a Docker setup.

> **Background:** I built this while following the *Python for Professionals* course. The structure and exercises come from the course material. I used it to learn how a backend API fits together, from the database layer to auth, testing, and Docker.

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Database Setup](#database-setup)
- [Running the Application](#running-the-application)
- [API Documentation](#api-documentation)
- [Testing and Code Quality](#testing-and-code-quality)
- [Docker](#docker)
- [Continuous Integration](#continuous-integration)
- [Known Limitations](#known-limitations)
- [Author](#author)

## Features

- **Projects:** create, list, get, update, and delete projects. Each project gets a URL-friendly `slug` generated from its name.
- **Tasks:** add tasks to a project with a status (`planned`, `in_progress`, `blocked`, `done`), a priority (`low`, `medium`, `high`, `urgent`), optional details, and an optional due date.
- **Task filtering:** filter tasks by project ID, project slug, status, priority, or show only overdue ones.
- **Authentication:** register and log in to get a JWT bearer token. Passwords are hashed with Argon2, and write operations need a valid token.
- **Validation:** request data is validated with Pydantic/SQLModel. For example, project names and task titles must be at least 2 characters after trimming whitespace.
- **Error handling:** missing resources return 404, and database conflicts like a duplicate project name return 409.
- **Health check:** a `/health` endpoint that checks the database connection.
- **Request logging:** a middleware logs the method, path, duration, and status code of every request.
- **Migrations:** database schema managed with Alembic.
- **Seed script:** loads sample data and a demo user for local development.
- **Tests:** 25 automated tests covering auth, projects, and tasks.
- **Tooling:** Ruff for linting and formatting, mypy for type checking, and coverage, all run in a GitHub Actions workflow.
- **Docker:** a Dockerfile plus Compose files for the app and the database.

## Tech Stack

| Area | Technology |
| --- | --- |
| Language | Python 3.14 |
| Web framework | FastAPI |
| ORM / models | SQLModel (SQLAlchemy + Pydantic) |
| Database | PostgreSQL 17 (psycopg 3 driver) |
| Migrations | Alembic |
| Authentication | PyJWT (HS256), pwdlib with Argon2 |
| Configuration | pydantic-settings |
| Package manager | uv |
| Testing | pytest, FastAPI TestClient (in-memory SQLite), coverage |
| Linting / typing | Ruff, mypy |
| Containers | Docker, Docker Compose |
| CI | GitHub Actions |

## Project Structure

```text
release-tracker-backend/
├── .github/workflows/ci.yml     # CI: lint, format check, type check, migrations, tests
├── alembic/                     # Alembic configuration and migration scripts
├── scripts/
│   └── seed.py                  # Loads sample data and a demo user
├── src/release_tracker/
│   ├── main.py                  # FastAPI app, middleware, exception handler
│   ├── config.py                # Settings loaded from environment / .env
│   ├── database.py              # Engine and session dependency
│   ├── models.py                # Database tables and request/response schemas
│   ├── crud.py                  # Database operations
│   ├── dependencies.py          # Shared dependencies (session, current user, 404 lookups)
│   ├── security.py              # Password hashing and JWT creation/validation
│   └── routers/
│       ├── auth.py              # /auth endpoints
│       ├── projects.py          # /projects endpoints
│       ├── tasks.py             # /tasks endpoints
│       └── health.py            # /health endpoint
├── tests/                       # pytest suite (auth, projects, tasks)
├── compose.yaml                 # PostgreSQL only (for local development)
├── compose.app.yaml             # API + PostgreSQL
├── Dockerfile
├── alembic.ini
├── pyproject.toml
├── ruff.toml
└── .env.example
```

## Getting Started

### Prerequisites

- **Python 3.14** (the project requires `3.14.*`)
- [**uv**](https://docs.astral.sh/uv/) for managing dependencies
- [**Docker**](https://docs.docker.com/get-docker/) with Docker Compose, to run PostgreSQL locally

### Installation

```bash
git clone https://github.com/trilochanms2035-cpu/release-tracker-backend.git
cd release-tracker-backend
uv sync
```

`uv sync` installs everything from `uv.lock` (dev tools included) into a local `.venv`.

## Environment Variables

Copy the example file and edit it:

```bash
cp .env.example .env
```

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `DATABASE_URL` | No | `postgresql+psycopg://release_tracker:release_tracker@localhost:5432/release_tracker` | SQLAlchemy connection string. The default matches `compose.yaml`. |
| `JWT_SECRET_KEY` | **Yes** | none | Secret used to sign JWTs. Must be at least 32 characters. |
| `DEBUG` | No | `false` | Set to `true` for debug-level logging. |

To generate a secret key, run this and paste the output into `.env`:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

`.env` is in `.gitignore`, so your secrets won't get committed.

## Database Setup

1. **Start PostgreSQL** (the credentials match the default `DATABASE_URL`):

   ```bash
   docker compose up -d db
   ```

2. **Create the tables with Alembic.**

   There are no migration files in this repo right now. `alembic/versions/` is empty, and `*.py` files in it are git-ignored. So generate the initial migration first, then apply it:

   ```bash
   uv run alembic revision --autogenerate -m "initial schema"
   uv run alembic upgrade head
   ```

   This creates the `projects`, `tasks`, and `users` tables.

3. **(Optional) Load sample data:**

   ```bash
   uv run python scripts/seed.py
   ```

   This adds 3 projects, 4 tasks, and a demo user:

   - Email: `demo@release-tracker.local`
   - Password: `demo-password`

   Only run it once. Running it again fails because of the duplicate user and project names.

## Running the Application

```bash
uv run fastapi dev
```

The API runs at <http://localhost:8000> and the interactive docs are at <http://localhost:8000/docs>. The app entrypoint (`release_tracker.main:app`) is set in `pyproject.toml`.

## API Documentation

FastAPI generates the docs automatically:

- Swagger UI: <http://localhost:8000/docs>
- ReDoc: <http://localhost:8000/redoc>

### Authentication

Protected endpoints need a bearer token. Get one from `/auth/token`, then send it in the `Authorization` header.

### Endpoints

| Method | Path | Auth | Description |
| --- | --- | --- | --- |
| GET | `/` | No | Basic app info and a link to the docs |
| GET | `/health` | No | Database health check (`200` healthy, `503` unhealthy) |
| POST | `/auth/register` | No | Register a user (`email`, `password`). Returns `201` |
| POST | `/auth/token` | No | Log in. Form fields: `username` (the email) and `password`. Returns a bearer token |
| GET | `/auth/me` | Yes | Current authenticated user |
| GET | `/projects/` | No | List projects (ordered by name) |
| GET | `/projects/{project_id}` | No | Get one project |
| POST | `/projects/` | Yes | Create a project. Returns `201` |
| PATCH | `/projects/{project_id}` | Yes | Partially update a project |
| DELETE | `/projects/{project_id}` | Yes | Delete a project. Returns `204` |
| GET | `/tasks` | No | List tasks, with optional filters (below) |
| GET | `/tasks/{task_id}` | No | Get one task |
| POST | `/projects/{project_id}/tasks` | Yes | Create a task in a project. Returns `201` |
| PATCH | `/tasks/{task_id}` | Yes | Partially update a task |
| DELETE | `/tasks/{task_id}` | Yes | Delete a task. Returns `204` |

**Task filters** for `GET /tasks`. All are optional and can be combined:

| Query parameter | Type | Description |
| --- | --- | --- |
| `project_id` | integer | Tasks belonging to a project ID |
| `project_slug` | string | Tasks belonging to a project slug |
| `status` | `planned` \| `in_progress` \| `blocked` \| `done` | Filter by status |
| `priority` | `low` \| `medium` \| `high` \| `urgent` | Filter by priority |
| `overdue_only` | boolean | Only tasks past their due date that are not `done` |

**Response codes:** `401` for a missing or invalid token or wrong credentials, `404` when a project or task doesn't exist, `409` for data conflicts like a duplicate project name, and `422` for validation errors.

### Example requests

Log in with the seeded demo user, then create a project and a task:

```bash
# 1. Log in and capture the token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
  -d "username=demo@release-tracker.local&password=demo-password" \
  | python -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# 2. Create a project
curl -X POST http://localhost:8000/projects/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Mobile App", "description": "Release planning for the mobile app"}'

# 3. Create a task in that project (use the id returned above)
curl -X POST http://localhost:8000/projects/4/tasks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "Design login screen", "priority": "high", "due_date": "2026-12-31"}'

# 4. List blocked tasks that are overdue
curl "http://localhost:8000/tasks?status=blocked&overdue_only=true"
```

Example task response:

```json
{
  "title": "Design login screen",
  "details": null,
  "status": "planned",
  "priority": "high",
  "due_date": "2026-12-31",
  "id": 5,
  "project_id": 4,
  "project_name": "Mobile App",
  "project_slug": "mobile-app"
}
```

## Testing and Code Quality

The tests use an in-memory SQLite database, so you don't need PostgreSQL running to execute them.

```bash
# Run the test suite (25 tests)
uv run pytest

# Run with coverage
uv run coverage run -m pytest
uv run coverage report

# Lint, format check, and type check
uv run ruff check .
uv run ruff format --check .
uv run mypy .
```

The tests cover login and registration, token validation, protected routes, project CRUD, task CRUD, task filtering, and error responses (401, 404, 409).

## Docker

`compose.app.yaml` builds the API image and runs it alongside PostgreSQL. When the API container starts, it runs `alembic upgrade head` and then `fastapi run`.

1. Make sure a migration exists in `alembic/versions/` before building, since the image copies that folder from your working directory. See [Database Setup](#database-setup) for how to generate one. If you were using `compose.yaml` for local development, stop it first so the ports don't clash:

   ```bash
   docker compose down
   ```

2. Build and start the app (use your own secret):

   ```bash
   export JWT_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(32))")
   docker compose -f compose.app.yaml up --build
   ```

3. Open <http://localhost:8000/docs>.

`JWT_SECRET_KEY` is required, and Compose exits with an error if it isn't set. The container health check calls `/health`.

## Continuous Integration

The GitHub Actions workflow (`.github/workflows/ci.yml`) runs on pushes to `main` and on pull requests. It starts a PostgreSQL service and then runs:

1. Dependency install (`uv sync --frozen`)
2. Lint (`ruff check`) and format check (`ruff format --check`)
3. Type check (`mypy`)
4. Database migrations (`alembic upgrade head`)
5. Tests with coverage

## Known Limitations

This is a learning project, so a few things are kept simple on purpose:

- **No per-user ownership.** Any logged-in user can update or delete any project or task.
- **Open registration** at `/auth/register`, with no password strength rules and no email verification.
- **No refresh tokens.** Access tokens expire after 60 minutes, which is a fixed value in `security.py`.
- **No pagination** on the list endpoints.
- **Not deployed.** There's no live deployment, and the Docker setup is meant for local use.
- **No license** file yet.

## Author

**Trilochan**

- GitHub: [trilochanms2035-cpu](https://github.com/trilochanms2035-cpu)
- LinkedIn: [trilochan-ms](https://www.linkedin.com/in/trilochan-ms-20019a2ab/)
