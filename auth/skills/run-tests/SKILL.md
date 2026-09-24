---
name: run-tests
description: Run the pesu-auth test suites correctly - targeted pytest, the canonical scripts/run_tests.py with its 95% coverage gate, the single-session live tests, and the full pre-commit run - and report results truthfully, including partial runs. Use whenever you need to know if the code works.
---

# Running tests

## Commands

| Goal                             | Command                                                                            |
| -------------------------------- | ---------------------------------------------------------------------------------- |
| One test                         | `uv run pytest tests/unit/test_pesu.py::test_name -v`                              |
| One file / keyword               | `uv run pytest tests/unit/test_request_model.py -q` / `uv run pytest -k fields -q` |
| Offline suite                    | `uv run pytest tests/unit -q`                                                      |
| Everything without credentials   | `uv run pytest -m "not secret_required" -q` (needs internet)                       |
| **Canonical run** (what CI runs) | `uv run python scripts/run_tests.py`                                               |
| Live tests only                  | `uv run pytest -m secret_required -v`                                              |
| All CI checks incl. tests        | `uv run pre-commit run --all-files`                                                |

`scripts/run_tests.py` loads `.env`, runs pytest with `--cov=app --cov-fail-under=95 --cov-report=term-missing`, and deselects the live tests (printing **"Live PESU tests skipped"**)
unless `TEST_EMAIL`, `TEST_PRN`, `TEST_PHONE` and `TEST_PASSWORD` are all set.

## The live-test rule

The test account allows **one session**. Before any run that includes live tests:

- make sure no other test run (yours, another agent's, a terminal) is in progress;
- never pass `-n`/`--numprocesses`, never run two pytest processes at once;
- avoid running right after a merge to `dev` (CI is using the account).

A live test failing with 401 on valid credentials is almost always a concurrent session: wait, then
rerun **once**.

## Interpreting results

- **Coverage:** the gate is 95%, but the suite is at 100%. A drop is a finding even when the gate
  passes; add tests for the lines listed under `Missing`.
- **Ordering:** `tests/conftest.py` forces unit → functional → integration. A test that passes alone
  but fails in the suite usually leaks state through a module singleton (`app.app.metrics`,
  `app.app.pesu_academy`, `app.openapi_schema`). Fix the isolation, not the order.
- **Network failures** in functional/integration tests (connection errors, timeouts): PESU Academy
  or the network is down. Not a code failure; say so and rely on `tests/unit` plus a later rerun.
- **One `DeprecationWarning`** from Starlette's test client is expected.

## Reporting

Always report the exact command and its summary line, for example:

> `uv run python scripts/run_tests.py` → 265 passed, 11 deselected, coverage 100%. **Partial run:**
> live PESU tests skipped (no credentials).

Never describe a partial run as a full pass, never omit failures, and never paste credential values
or personal data from live-test output.
