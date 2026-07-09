# IS218 Module 10 — Secure User Model & CI/CD

FastAPI application with a SQLAlchemy `User` model, Pydantic validation
schemas, bcrypt password hashing, and a GitHub Actions pipeline that runs
the test suite against a real Postgres database and pushes a Docker image
to Docker Hub on success.

## Docker Hub

Image: **https://hub.docker.com/r/hackandquack/601_module10**

```bash
docker pull hackandquack/is218-module10-user-api:latest
docker run -p 8000:8000 hackandquack/is218-module10-user-api:latest
```

## Secure User Model

- `app/models/user.py` — SQLAlchemy `User` model: unique `username` and
  `email`, a `password_hash` column (never the raw password), a
  `created_at` timestamp, and bcrypt hashing via `passlib`
  (`User.hash_password` / `user.verify_password`).
- `app/schemas/base.py` — `UserCreate` (username, email, password) with
  password-strength validation.
- `app/schemas/user.py` — `UserRead` (returns user details, omits
  `password_hash`).

## Running Tests Locally

Tests need a Postgres database (the integration tests exercise real
uniqueness/constraint behavior, so SQLite won't do).

**1. Start Postgres** (either works):

```bash
# Option A: docker-compose (spins up Postgres + pgAdmin)
docker compose up -d db

# Option B: a local Postgres instance — just make sure a DB exists that
# matches DATABASE_URL below
```

**2. Install dependencies:**

```bash
python3 -m venv venv
source venv/bin/activate        # venv\Scripts\activate on Windows
pip install -r requirements.txt
playwright install --with-deps chromium   # only needed for e2e tests
```

**3. Set the database URL and run the suite:**

```bash
export DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fastapi_test_db

pytest tests/unit/            # unit tests: hashing, schema validation
pytest tests/integration/     # integration tests: real DB, uniqueness, constraints
pytest tests/e2e/             # end-to-end (starts the app + drives it with Playwright)

# or everything at once, with coverage (as configured in pytest.ini)
pytest
```

Useful flags: `--preserve-db` keeps the test data instead of truncating it
after each test; `--run-slow` includes tests marked `@pytest.mark.slow`.

## Running the App

```bash
# without Docker
uvicorn main:app --reload

# with Docker Compose (app + Postgres + pgAdmin)
docker compose up
```

The app serves a calculator UI at `/`, calculator endpoints at
`/add`, `/subtract`, `/multiply`, `/divide`, and a `/health` check used by
the Docker `HEALTHCHECK` and CI pipeline.

## CI/CD Pipeline (`.github/workflows/test.yml`)

1. **test** — spins up a Postgres service container, installs
   dependencies, runs unit/integration/e2e tests.
2. **security** — builds the Docker image and scans it with Trivy for
   CRITICAL/HIGH vulnerabilities.
3. **deploy** — on pushes to `main`, after `test` and `security` pass,
   builds a multi-arch image and pushes it to Docker Hub using the
   `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` repository secrets.

---

## Local Environment Setup (Git/Python/Docker)

<details>
<summary>Expand for first-time machine setup instructions</summary>

### Install Homebrew (Mac Only)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install and Configure Git

```bash
brew install git          # or Git for Windows: https://git-scm.com/download/win
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

Generate an SSH key and add it to GitHub if you haven't already:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub | pbcopy   # then add it at https://github.com/settings/keys
ssh -T git@github.com
```

### Clone the Repository

```bash
git clone <repository-url>
cd <repository-directory>
```

### Install Python 3.10+

```bash
brew install python   # or https://www.python.org/downloads/ on Windows
python3 --version
```

### Docker

- [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/)
- [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)

</details>
