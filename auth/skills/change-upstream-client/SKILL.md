---
name: change-upstream-client
description: Safely change pesu-auth's upstream client machinery in app/pesu.py and the lifespan - CSRF prefetch cache, locking, background tasks, HTTP client creation and closing, timeouts, cancellation - without reintroducing connection leaks, races or lost exceptions. Use for any change to concurrency, client lifecycle or upstream request handling.
---

# Changing the upstream client

This code has had real leaks and races; nearly every line in it exists because of one. Read
"The CSRF prefetch design" and "Client lifecycle and cancellation" in
`.agents/pesu-skills/auth/docs/architecture.md`, and the comments in `app/pesu.py`, before editing.

## Invariants you must preserve

1. **Every `httpx2.AsyncClient` is closed exactly once, on every path**: success, exception,
   `CancelledError`, and cancellation arriving *during* cleanup. Use `_close_client_quietly` (shielded,
   strong-referenced) for closes in `except BaseException`/`finally` blocks.
1. **A client is used by exactly one login.** Take it out of the cache under `_csrf_lock` and clear
   the cache in the same critical section.
1. **Never hold `_csrf_lock` across an upstream request.** Fetch outside, swap inside.
1. **Background tasks are strong-referenced** (`_prefetch_tasks`, `_CLOSE_TASKS`) and their results
   retrieved in a done callback, so failures are logged and counted instead of "Task exception was
   never retrieved".
1. **Shutdown cancels prefetches before closing the cached client** (`close_client`), so nothing is
   cached after close.
1. **Every upstream call is inside `_upstream_call`** with an operation label, so it is counted and
   timed exactly once; cancellation is `cancelled`, not `error`.
1. **Cleanup failures never replace the original error** (a failing close must not turn a 401 into a
   500).
1. **HTML parsing runs in `asyncio.to_thread`.**
1. `except BaseException` where cancellation must trigger cleanup; `except Exception` where it must
   not be swallowed. Always re-raise.

## Testing each path

For every path you touch, write a test that fails if the invariant breaks. Existing examples in
`tests/unit/test_pesu.py` to copy:

- `test_authenticate_closes_client_on_*`: close on success and on each error.
- `test_fetch_new_client_closes_client_when_*`: close when the fetch fails.
- `test_prefetch_closes_new_client_when_cancelled_before_caching` and
  `..._during_cleanup`: cancellation at each await.
- `test_close_client_cancels_in_flight_prefetch`, `test_get_client_holds_strong_reference_to_prefetch_task`,
  `test_prefetch_task_failure_is_logged`, `test_authenticate_triggers_exactly_one_prefetch`.

Technique: patch `httpx2.AsyncClient` (or `_fetch_new_client_with_csrf_token`) with `AsyncMock`s,
control ordering with `asyncio.Event`s, cancel with `task.cancel()` at a chosen await, then assert
`client.aclose.await_count == 1` and the `http_clients_total` created/closed counts on a fresh
collector.

## Verifying for real

```bash
uv run python scripts/run_tests.py
uv run pytest -m secret_required -v                  # once, alone
uv run python -m app.app                             # then, in another shell:
curl -s "localhost:5000/metrics?fmt=json" | python -m json.tool | grep -A4 httpClients
```

After some logins and at rest, `created - closed` must be **1**. Stop the server with Ctrl+C and
check the log shows a clean shutdown with no "Task was destroyed" or "Unclosed client" warnings. For
load behaviour, run the `benchmark` skill with `--parallel` against the local server (small request
counts: it logs in to PESU for real) and check `csrfCache` hit rate and the client counts again.

## Changing the refresh interval or timeouts

`CSRF_TOKEN_REFRESH_INTERVAL_SECONDS` (45 min) should stay below the token's lifetime; measure it
with `scripts/benchmark/unauthenticated_csrf_token_expiry.py` before changing it. The client timeout
(10s) bounds how long a caller can wait on PESU; changing it changes user-visible latency and should
be justified in the PR.
