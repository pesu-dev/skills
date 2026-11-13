---
name: fix-ci
description: Diagnose and fix failing GitHub Actions checks on a pesu-auth PR or branch - the CI Checks jobs (PR source checker, Pre-commit hooks, Test suite & coverage, Build image on PR, Container security & smoke test, Version check, Greenlight) - by reading the failed logs, reproducing locally and fixing the cause without weakening any check. Use whenever a check is red.
---

# Fixing CI

Workflow reference: `.agents/pesudev-skills/auth/docs/ci-cd.md`.

## 1. Find the failure

```bash
gh pr checks <n> --repo pesu-dev/auth
gh run list --repo pesu-dev/auth --branch <branch> --limit 5      # runs on the PR's head
gh run view <run-id> --repo pesu-dev/auth --log-failed | head -200
```

*Greenlight - CI checks* only collects the other jobs' results: find the job that actually failed.
Read the first real error in its log, not the summary at the bottom. Redact anything sensitive
before quoting logs.

## 2. Diagnose by job

| Job                                 | Typical causes                                                                                                                                                                                  | Reproduce and fix                                                                                                                                                                                                                                                                                                   |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PR source checker**               | The PR comes from a fork, its head branch is `dev` or `main`, or it targets something other than `dev`                                                                                          | Fix the PR: a PR's head cannot be changed, so for a fork or a `dev`/`main` head push the commits to a new branch of `pesu-dev/auth` and open a new PR from it (close the old one); for a wrong base, `gh pr edit <n> --repo pesu-dev/auth --base dev`                                                               |
| **Pre-commit hooks**                | ruff lint or format, mdformat rewrote a Markdown file, trailing whitespace or EOF, `uv-lock` (the lock no longer matches `pyproject.toml`), `name-tests-test`, `debug-statements`, broken links | `uv run pre-commit run --all-files`; commit what the fixers changed; `uv lock` for a stale lock. CI runs every hook but `pytest` here                                                                                                                                                                               |
| **Pre-commit hooks: `sync-agents`** | `.github/agents/` no longer matches the roles in `.agents/pesudev-skills/auth/agents/`: usually a submodule bump that changed a role's frontmatter, or someone edited a wrapper by hand         | On your own PR: `uv run python scripts/sync_agents.py` and commit the regenerated `.github/agents/`. If it says the submodule is missing, run `git submodule update --init` first. On a Dependabot PR do not push to its branch: follow "Dependabot submodule bumps" in `.agents/pesudev-skills/auth/docs/ci-cd.md` |
| **Test suite & coverage**           | A failing test, or coverage below 100% (line or branch). The live tests run here too (not on Dependabot's PRs): a field mismatch usually means PESU's API changed or a `TEST_*` secret is stale | `uv run python scripts/run_tests.py`; read the `Missing` column and cover those lines and branches                                                                                                                                                                                                                  |
| **Build image on PR**               | The image does not build: the lock is out of date (`uv sync --locked` refuses it), or a runtime file outside `app/` is not copied                                                               | `docker build . --tag pesu-auth-test`                                                                                                                                                                                                                                                                               |
| **Container security & smoke test** | The container does not run as UID 10001, or does not answer `/health` within a few seconds (a startup crash, such as an import error)                                                           | the Docker steps in `run-locally`; `docker logs` shows the startup error                                                                                                                                                                                                                                            |
| **Version check**                   | The version did not go up by exactly one step from the base (unchanged, skipped one, or not reset to its right), or `uv.lock` was not updated                                                   | `bump-version` (includes the exact local command)                                                                                                                                                                                                                                                                   |

Failures in `tests/functional/` or `tests/integration/` with connection errors or timeouts usually
mean PESU Academy was unreachable from the runner. Rerun once:
`gh run rerun <run-id> --repo pesu-dev/auth --failed` (needs permission on the repo; if you lack it,
push an empty change only if the human agrees, or report it).

## 3. Fix and push

Fix on the PR branch, commit with the right type (`style:`, `fix:`, `test:`, `chore:`), run
`uv run pre-commit run --all-files` locally, push the branch. Checks re-run automatically.

## Never

Edit `.github/workflows/*`, `.pre-commit-config.yaml`, the coverage gate or ruff configuration to
make a failure go away; skip hooks with `--no-verify`; mark tests `skip`/`xfail` to get green. If a
check itself is wrong, say so in the report and leave the decision to a maintainer.

Three attempts on the same failure without progress → stop and report with the log excerpt.
