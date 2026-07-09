# Reflection — Module 10: Secure User Model & CI/CD

## What I built

Starting from the calculator app used in earlier modules, I added a
security-focused `User` layer: a SQLAlchemy model with a `password_hash`
column and unique constraints on `username`/`email`, Pydantic `UserCreate`
and `UserRead` schemas, bcrypt-based hashing and verification, and unit +
integration tests that exercise both the pure-Python validation logic and
real database constraint behavior against Postgres.

## Key experiences

**Separating the input shape from the storage shape.** The `UserCreate`
schema accepts a plaintext `password` field — that's the correct name for
an API input. The database, however, should never hold that value; it
should only ever hold the bcrypt digest. Renaming the model column from
`password` to `password_hash` made that boundary explicit in the code
itself rather than just in a docstring, and it forced me to trace every
place a `User` gets constructed (fixtures, direct test instantiation, the
`register()` classmethod) to make sure nothing was quietly passing a raw
password into a column that's supposed to hold a hash.

**Tests as a safety net for a refactor.** Because the existing test suite
already covered uniqueness constraints, rollback behavior, and the
registration/authentication flow fairly thoroughly, the column rename was
low-risk: I could make the change and immediately see which of the 72
tests broke (four did, all from tests constructing `User` objects
directly rather than through `register()`), fix them, and get back to a
green suite. That's the value of integration tests against a *real*
database instead of mocks — a SQLite stand-in wouldn't have caught the
same constraint/rollback behavior.

## Challenges

**The Docker Hub target was pointed at someone else's namespace.** The
CI/CD workflow I inherited hardcoded `kaw393939/601_module9` as the image
tag — the instructor's own Docker Hub repository, not mine. This is an
easy thing to miss if you only check that the pipeline runs green,
because the build and push steps succeed either way; they just push to
the wrong place (and would fail outright once Docker Hub credentials
didn't match). I changed the tag to build from
`${{ secrets.DOCKERHUB_USERNAME }}` instead of a hardcoded name, so the
image always lands in whichever account owns the `DOCKERHUB_USERNAME` /
`DOCKERHUB_TOKEN` secrets configured on the repo.

**A health check that couldn't succeed.** The `Dockerfile` had a
`HEALTHCHECK` calling `curl http://localhost:8000/health`, but the image
never installed `curl`, and the app never defined a `/health` route. The
container would have started successfully and then been marked unhealthy
by Docker itself — a failure mode that's invisible unless you actually
inspect container health, not just "does `docker run` exit 0." I added
the missing route and the missing package.

**Coverage misconfiguration in CI.** The test job ran
`pytest tests/unit/ --cov=src`, but the source lives in `app/`, not
`src/` — coverage was silently measuring nothing. Small, easy to miss,
but it meant the coverage numbers reported in earlier CI runs weren't
measuring the actual codebase.

## What I'd do differently next time

I'd wire the `User` model into actual FastAPI routes (`/register`,
`/token`, `/users/me`) in this module rather than leaving it as a
model/schema layer with no HTTP surface — right now the auth
infrastructure (JWT creation, `get_current_user`) exists but nothing in
`main.py` calls it. That's the natural next step for the following
module, building on this same `User` model and hashing logic.
