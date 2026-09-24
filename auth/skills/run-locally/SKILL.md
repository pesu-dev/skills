---
name: run-locally
description: Run pesu-auth locally from source or as the Docker image and smoke-test every endpoint (Swagger, /health, /authenticate, /metrics with and without METRICS_TOKEN, /readme), reproducing what the Docker CI workflow checks. Use to verify a change end to end, reproduce a bug, or check the image builds.
---

# Running pesu-auth locally

## From source

```bash
uv run python -m app.app --debug          # http://localhost:5000, DEBUG logs, auto-reload
uv run python -m app.app --port 5001      # another port
METRICS_TOKEN=local-secret uv run python -m app.app   # with /metrics protected
```

Startup awaits a CSRF prefetch from pesuacademy.com in `lifespan`, so **without access to
pesuacademy.com the app does not start** (and neither does the Docker smoke test). For offline work,
use `TestClient` with the prefetch patched (`write-tests`). Run the server in the background or a
second shell, and stop it when you are done.

## Smoke test

```bash
curl -s -o /dev/null -w "%{http_code}\n" localhost:5000/                  # 200 Swagger UI
curl -s localhost:5000/health                                             # {"status":true,"message":"ok",...}
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" localhost:5000/readme   # 308 https://github.com/pesu-dev/auth
curl -s localhost:5000/metrics | head -5                                  # Prometheus text
curl -s "localhost:5000/metrics?fmt=json" | python -m json.tool | head -20
curl -s "localhost:5000/metrics?fmt=xml"                                  # 400, standard error body
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d '{"username": "", "password": "x"}'                                  # 400 validation body
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d '{"username": "INVALID_USER", "password": "wrong"}'                  # 401 (real PESU call)
```

A real login with the test account (one session; do not print the response's profile values into
logs or reports):

```bash
set -a; . ./.env; set +a
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d "{\"username\": \"$TEST_PRN\", \"password\": \"$TEST_PASSWORD\", \"profile\": true}" \
  | python -c 'import json,sys; b=json.load(sys.stdin); print(b["status"], sorted(b.get("profile", {})))'
```

With `METRICS_TOKEN` set: `/metrics` without a header is 401 with `WWW-Authenticate: Bearer`
(`curl -si`), and `-H "Authorization: Bearer local-secret"` is 200.

## Docker (what `docker.yaml` checks)

```bash
docker build . --tag pesu-auth
docker run --rm --name pesu-auth -d -p 5000:5000 pesu-auth
sleep 5 && curl --fail --max-time 10 http://localhost:5000 > /dev/null && echo OK
docker logs pesu-auth | tail -20
docker stop pesu-auth
```

Run it whenever the change touches runtime code paths used at startup, dependencies, `uv.lock` or
the `Dockerfile`. The image copies only `pyproject.toml`, `uv.lock`, `README.md` and `app/`; a new
runtime file outside `app/` must be added to the `Dockerfile`.

## Report

List each request with its status and the relevant part of the body (never credentials or personal
data), and whether Docker was built and answered.
