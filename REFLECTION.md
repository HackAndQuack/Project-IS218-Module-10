# Reflection — Module 10: Secure User Model & CI/CD

## What I built

Starting from the calculator app used in earlier modules, I added a
security-focused `User` layer: a SQLAlchemy model with a `password_hash`
column and unique constraints on `username`/`email`, Pydantic `UserCreate`
and `UserRead` schemas, bcrypt-based hashing and verification, and unit +
integration tests that exercise both the pure-Python validation logic and
real database constraint behavior against Postgres. On top of that, I
had to actually get the thing running and deployed, which turned out to
be where most of the real learning happened.

## Key experiences

**Separating the input shape from the storage shape.** The `UserCreate`
schema accepts a plaintext `password` field — that's the correct name for
an API input. The database should never hold that value; it should only
ever hold the bcrypt digest. Renaming the model column from `password` to
`password_hash` made that boundary explicit in the code itself rather
than just in a docstring, and it forced me to trace every place a `User`
gets constructed (fixtures, direct test instantiation, the `register()`
classmethod) to make sure nothing was quietly passing a raw password into
a column that's supposed to hold a hash.

**A security scan is not optional busywork — it found real bugs.**
Running Trivy against the built image turned up 18 vulnerabilities (2
CRITICAL, 16 HIGH) across dependencies like `starlette`, `python-jose`,
`h11`, and `urllib3`. Fixing these wasn't just bumping version numbers —
it was a small dependency-resolution puzzle. `python-jose` 3.4.0 still
pins `pyasn1<0.5.0`, which directly conflicts with the fix for `pyasn1`'s
own CVE (which requires `>=0.6.2`); only 3.5.0 relaxes that constraint.
Bumping `h11` to close its CRITICAL CVE broke `httpcore`, which pinned
`h11<0.15`. And closing the `starlette` CVEs required a major-version
jump (0.41 → 1.3.1), which only worked with a compatible `fastapi`
version. None of this was visible from the scan output alone — it took
actually attempting the install and reading the resulting dependency
conflicts to work out a version set that was simultaneously secure and
mutually compatible.

**Passing tests don't mean the app works.** The dependency bumps passed
the full test suite (71/71) on the first try — and were still broken.
The `starlette` major version changed `Jinja2Templates.TemplateResponse`'s
signature from `(name, {"request": request})` to `(request, name)`. The
old call site would have crashed the `/` route in production with
`TypeError: unhashable type: 'dict'`. No test covered the `/` route
directly (only the JSON calculator endpoints and the Postgres-backed user
model had integration tests) — I only caught it by manually hitting the
route with a test client and reading the actual rendered response, which
is a good reminder that coverage percentage isn't the same as "the app
still works."

## Challenges — deployment

This is where things got genuinely hard, and where I learned the most.

**Docker Hub was pointed at someone else's account.** The CI/CD workflow
I inherited hardcoded `kaw393939/601_module9` as the deploy target — the
instructor's own Docker Hub namespace. It's an easy thing to miss because
the pipeline still runs green; it just deploys to the wrong place. Fixed
by tagging from `${{ secrets.DOCKERHUB_USERNAME }}` instead of a literal
string, so the image always lands wherever the repo's own secrets point.

**A health check that could never succeed.** The Dockerfile's
`HEALTHCHECK` called `curl http://localhost:8000/health`, but the image
never installed `curl` and the app never defined that route. The
container would start fine and then silently be marked unhealthy —
invisible unless you actually inspect container health rather than just
checking that `docker run` exits 0.

**Postgres 18's breaking data-directory change.** Locally, `docker-compose.yml`
pinned nothing (`image: postgres`), which floats to `latest`. Postgres 18
changed how the official image expects its data directory to be laid out
(a single mount at `/var/lib/postgresql` instead of
`/var/lib/postgresql/data`, for `pg_upgrade`/`pg_ctlcluster` compatibility),
so an existing volume created by an older Postgres major version made the
container refuse to start entirely. The fix was two-fold: wipe the
now-incompatible volume, and pin the image to `postgres:16` so a routine
`docker compose pull` can't silently break the stack again the same way.

**Root inside a container isn't root on the host.** The last blocker was
the strangest: `docker compose exec web stat /app/main.py` returned
`Permission denied` even after switching the container to run as root.
Since root normally bypasses standard Unix permission checks entirely,
that ruled out ordinary file-mode issues and pointed at something
enforcing access control *underneath* the container — which turned out
to be SELinux. On an SELinux-enforcing host, bind-mounted files carry a
security context that's opaque to a container process regardless of
UID, and `chmod` does nothing to fix it because SELinux enforcement is
completely independent of the standard owner/group/other permission
bits. The actual fix was relabeling the mount (`:Z` in the compose volume
spec, or `chcon -Rt container_file_t` on the host) — a class of problem
that's invisible unless you already know SELinux exists, and one no
amount of `chmod -R 777` would ever have solved.

## What I'd do differently next time

I'd wire the `User` model into actual FastAPI routes (`/register`,
`/token`, `/users/me`) in this module rather than leaving it as a
model/schema layer with no HTTP surface — the auth infrastructure (JWT
creation, `get_current_user`) exists but nothing in `main.py` calls it
yet. I'd also add an integration test that actually hits `/` through a
test client, since that's the exact gap that let the `TemplateResponse`
regression slip past a fully green test suite.
