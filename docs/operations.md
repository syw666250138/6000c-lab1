# Operations Notes

## Local startup

1. copy `.env.example` to `.env`
2. run `docker compose up --build`
3. open the API docs if desired
4. submit a case through the API
5. confirm the worker processes the job

## Health endpoints

- API live: `/health/live`
- API ready: `/health/ready`
- AI live: `/health/live`
- AI ready: `/health/ready`

## Metrics

- API metrics: `/metrics`
- AI metrics: `/metrics`
- Prometheus: enable the `observability` profile

## Common useful commands

- run tests: `pytest -q`
- run smoke tests: `SMOKE_BASE_URL=http://localhost:8000 pytest tests/smoke -q`
- run migrations: `docker compose run --rm api alembic upgrade head`
- seed demo data: `docker compose run --rm api python scripts/seed_demo_data.py`

## Failure handling in the starter

If AI processing fails:

- the job is marked failed
- the case is marked failed
- the error is recorded on the job row

## Resetting local state

To remove local database state:

- `docker compose down -v --remove-orphans`


## Readiness probe

`GET /health/ready` returns `200` only when the API can execute a simple query
against PostgreSQL. It is used by Docker Compose as the API healthcheck.

`GET /health` remains the liveness check and does not depend on the database.
