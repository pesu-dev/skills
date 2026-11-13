# CI/CD and branching

How changes reach `dev`, what CI checks, and how releases happen. The repository's
`docs/deployment.md` describes the environments and the release steps for humans; this file is what
an agent needs to work within them.

## Branches and remotes

- `pesu-dev/auth` has two long-lived branches: **`dev`** (integration, deployed to staging) and
  **`main`** (production, only ever fast-forwarded to a released commit by the production workflow).
  The default branch is `dev`.
- Contributors clone `pesu-dev/auth` itself and push their branches to it
  (`.github/CONTRIBUTING.md`); forks are not accepted. `origin` is
  `https://github.com/pesu-dev/auth` (check with `git remote -v`). A clone from the old fork
  workflow has the fork as `origin` and pesu-dev/auth as `upstream`; fix it with
  `git remote rename origin fork && git remote rename upstream origin && git fetch origin && git branch -u origin/dev dev`.
- Feature branches are named for the change (`feat/metrics-bearer-token`, `fix/...`, `docs/...`),
  created from an up-to-date `origin/dev`. Everyone shares the branch namespace: pick a name nobody
  uses (`git ls-remote --heads origin <name>` prints nothing), and push only to your own branches.
- Nobody pushes to `dev` or `main` directly; changes land by PR into `dev`, squash-merged with a
  Conventional Commit title and `(#<PR number>)`.

## Workflows (`.github/workflows/`)

**CI Checks** (`ci_checks.yml`) runs on every PR, on pushes to `dev` and in the merge queue. Its jobs:

| Job                             | What it checks                                                                                                                                                                                                                                  | Reproduce locally                                                                                                                                                         |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PR source checker               | The PR comes from a branch of `pesu-dev/auth` (not a fork), its head branch is not `dev` or `main`, and its base is `dev`. Every other job waits for it                                                                                         | `gh pr view <n> --repo pesu-dev/auth --json isCrossRepository,headRefName,baseRefName`                                                                                    |
| Pre-commit hooks                | Checkout with the `.agents/pesudev-skills` submodule, `uv sync --locked`, then every hook in `.pre-commit-config.yaml` except `pytest` (`SKIP=pytest`): file hygiene, `mdformat`, ruff, `uv-lock`, `sync-agents`, test naming, debug statements | `uv run pre-commit run --all-files`                                                                                                                                       |
| Test suite & coverage           | `uv sync --locked --all-groups`, then `uv run python scripts/run_tests.py` with the 100% line and branch gate. The `TEST_*` secrets reach it on PRs, in the merge queue and on pushes to `dev`, so it runs the live tests every time            | `uv run python scripts/run_tests.py`                                                                                                                                      |
| Build image on PR               | PRs only: the image builds for the PR's head commit (`pesu-dev/actions` `build_push_image.yml`, `push: false`)                                                                                                                                  | `docker build . --tag pesu-auth-test`                                                                                                                                     |
| Container security & smoke test | The image runs as UID 10001 and answers `/health` within a few seconds of starting                                                                                                                                                              | the Docker steps in the `run-locally` skill                                                                                                                               |
| Version check                   | `.github/scripts/check_version_bump.py`: `project.version` is exactly one step above the base's, and `uv.lock` records the same version. Skipped for Dependabot                                                                                 | `python3 .github/scripts/check_version_bump.py --base-pyproject <(git show origin/dev:pyproject.toml) --head-pyproject pyproject.toml --head-lock uv.lock --base-ref dev` |
| Greenlight - CI checks          | Fails if any job above failed or was cancelled (a skipped job passes)                                                                                                                                                                           | none                                                                                                                                                                      |

The other workflows:

| Workflow                                        | Trigger                                                         | What it does                                                                                                                                                                                   |
| ----------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Copilot Setup Steps (`copilot-setup-steps.yml`) | pushes and PRs that change it, manual dispatch                  | The environment GitHub Copilot's cloud agent starts in: checkout with the submodule, Python 3.14, `astral-sh/setup-uv`, `uv sync --locked --all-groups`, `contents: read`. No `TEST_*` secrets |
| Deploy to Staging (`deploy_staging.yml`)        | after CI Checks succeeds on a push to `dev`, or manual on `dev` | Skips if `dev` moved past the validated commit; otherwise deploys that commit to Render staging (`pesu-dev/actions` `deploy_render.yml`) and waits for it to go live                           |
| Deploy to Production (`deploy_prod.yml`)        | manual, on `dev` only                                           | Checks, approval in the `promote-gate` environment, deploy with automatic rollback, release, then `main` fast-forwarded. See "Deployment flow"                                                 |

Dependabot (`.github/dependabot.yml`) opens a PR against `dev` daily when pesu-dev/skills `main`
has moved.

## Dependabot submodule bumps

Dependabot's PRs come from a branch in `pesu-dev/auth` itself, like everyone's, so the PR source
checker accepts them, and the Version check skips them.

If the bump adds or removes a role or changes a role's frontmatter, `sync-agents` fails on it,
because `.github/agents/` is out of date. Dependabot cannot fix that. Either:

- a maintainer checks out the Dependabot branch, runs `uv run python scripts/sync_agents.py`, and
  pushes the regenerated `.github/agents/` to it; or
- anyone opens a pull request from a new branch that bumps the submodule to the same commit
  (`git -C .agents/pesudev-skills checkout <sha>`, then `git add .agents/pesudev-skills`), runs the
  script, commits `.github/agents/`, and bumps the version as for any pull request. Dependabot
  closes its own pull request once `dev` already has that commit.

Agents take the second route: they push only to branches they created. A bump that only changes
role bodies, skills or docs passes as is.

## Deployment flow

1. PR opened or updated → CI Checks on the PR (with the live tests), and Render builds a preview of
   it at `https://pesuauth-dev-pr-<n>.onrender.com`. A preview runs the PR's unreviewed code with
   pesuauth-dev's environment variables, so its `/health` reports `staging`; it is not staging, and
   it is removed when the PR closes.
1. PR merged into `dev` (through the merge queue, which runs CI Checks with the live tests again) →
   CI Checks on `dev` → Deploy to Staging (`https://pesuauth-dev.onrender.com`). Render builds the
   commit itself; staging publishes no image.
1. A maintainer validates staging, then starts Deploy to Production on `dev`. It releases `dev`'s
   latest commit:
   1. **Checks:** CI Checks passed on that commit's push to `dev`, the version is newer than the
      latest `v*` tag, and the Docker Hub credentials are set.
   1. **Approval:** a reviewer of the `promote-gate` environment approves.
   1. **Deploy:** the commit live on production is recorded, the image is built for `linux/amd64`
      and `linux/arm64` and pushed to GHCR as `:<sha>`, and Render production deploys the commit. If
      that fails, the recorded commit is redeployed and the run is marked failed.
   1. **Release:** the image is tagged `vX.Y.Z`, `latest` and `prod` on GHCR
      (`ghcr.io/pesu-dev/pesu-auth`) and Docker Hub (`pesudev/pesu-auth`), and the git tag and
      GitHub Release are created.
   1. **Update `main`:** `main` is fast-forwarded to the released commit.
1. `/health` shows the deployed `version` and its `environment` (`production` or `staging`), and
   `/openapi.json` shows the version as `info.version`.

A run that fails after the deploy has already put the new commit on production; a maintainer fixes
the cause and re-runs the failed jobs. Agents perform none of this.

## Things agents must never do

- Push to `dev` or `main`, or to a branch they did not create; delete branches; merge PRs; approve
  PRs.
- Start Deploy to Staging or Deploy to Production, approve `promote-gate`, or call Render.
- Edit secrets, repository variables or environment settings.
- Change workflows to skip or weaken a check to get a PR green.

## Reading CI results

```bash
gh pr checks <pr-number> --repo pesu-dev/auth
gh run list --repo pesu-dev/auth --branch <branch> --limit 5
gh run view <run-id> --repo pesu-dev/auth --log-failed
```

See the `fix-ci` skill for failure-by-failure diagnosis.
