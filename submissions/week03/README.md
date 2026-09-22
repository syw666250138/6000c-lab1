• My bounded change: add a database-backed readiness probe
• Why it is useful: It improves observability without changing the core workflow. It verifies that the API can reach PostgreSQL before being considered ready, while keeping the existing liveness endpoint separate.
• Files I expect to inspect/change: 
- Added `GET /health/ready` readiness endpoint in `services/api/app/routes/health.py` that runs `SELECT 1` against PostgreSQL and returns `200` with `{"status": "ok", "checks": {"database": "ok"}}`, or `503` with `{"detail": "database unavailable"}` when the query fails.
- Registered the health router in `services/api/app/main.py`, keeping the existing `/health` endpoint as the liveness probe.
- Updated the `api` healthcheck in `compose.yaml` to call `/health/ready` via `python -c`.
- Added readiness integration coverage in `tests/integration/test_health.py`.  
• Behaviour before: 
- Only a liveness endpoint existed (`/health`). It returned 200 whenever the API
    process was up, regardless of whether PostgreSQL was reachable.
- Docker Compose treated the API as healthy as soon as the process answered,
    so a container with a broken DB connection could still report `healthy`.
- No automated test asserted anything about DB connectivity through a health route.
• Behaviour after:
- New `GET /health/ready` readiness endpoint executes `SELECT 1` against
    PostgreSQL and returns:
      200 {"status":"ok","checks":{"database":"ok"}}   when the DB answers
      503 {"detail":"database unavailable"}             when the query fails
- `/health` remains unchanged as the liveness probe, so liveness and readiness
    now have distinct semantics.
- Docker Compose `api` healthcheck points at `/health/ready` (via `python -c`,
    no `curl` needed in the image), so `healthy` now implies DB reachability.
- Integration coverage in `tests/integration/test_health.py` asserts the 200
    path and the 503 path (DB dependency overridden to raise).
• Verification (test / curl / logs / SQL /
clean clone):
- curl:
      curl -fsS http://localhost:8000/health/ready
      # => {"status":"ok","checks":{"database":"ok"}}
- docker compose:
      docker compose up --build -d
      docker compose ps
      # api transitions to (healthy)
- tests:
      docker compose run --rm --no-deps api pytest -q
      SMOKE_BASE_URL=http://localhost:8000 pytest tests/smoke -q
• Failure or rollback plan (git switch main;
branch can be discarded):
- This change is isolated to one feature branch. If it breaks anything:
      docker compose down --remove-orphans
      git switch main
      # health-readiness can be discarded:
      git branch -D health-readiness

AI tool name : Deepseek
what it was used for : Code check 
what you verified, changed, or rejected : fix compile error in health.py

