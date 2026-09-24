---
name: bump-version
description: Bump the pesu-auth project version exactly once per pull request - minor by default, major for breaking changes - update uv.lock to match, and verify with the same script the version-check workflow runs. Use when preparing any PR to pesu-dev/auth.
---

# Bumping the version

`version-check.yaml` fails a PR unless `project.version` in `pyproject.toml` is strictly greater
than on the base branch **and** `uv.lock` records the same version. The version is the API version
shown in `/openapi.json` and on the README badges.

## Rules (from `.github/scripts/check_version_bump.py`)

- **One bump per PR**, relative to `upstream/dev`, not per commit or feature.
- **Minor** (`x.Y.0`, raise the minor, reset patch to 0) for everything by default: features, fixes,
  docs, refactors, dependencies.
- **Major** (`X.0.0`) only for backwards-incompatible API or schema changes
  (`.agents/pesu-skills/auth/docs/api-contract.md`).
- Patch releases are not used in practice.

## Steps

```bash
git fetch upstream
git show upstream/dev:pyproject.toml | grep -m1 '^version'   # base version, e.g. version = "4.7.0"
grep -m1 '^version' pyproject.toml                            # yours
```

If your branch already bumps above the base (for example after a rebase where `dev` did not move),
do nothing. If `dev` moved and now equals or passes your version, bump again from the new base.

Edit `version = "X.Y.Z"` in the `[project]` table of `pyproject.toml`, then:

```bash
uv lock                                       # records the new version for pesu-auth in uv.lock
python3 .github/scripts/check_version_bump.py \
  --base-pyproject <(git show upstream/dev:pyproject.toml) \
  --head-pyproject pyproject.toml --head-lock uv.lock --base-ref dev
git add pyproject.toml uv.lock
git commit -m "chore: bump version to X.Y.Z"
```

The check must print `✅ Version raised from ... to ..., with uv.lock in step.`

`uv lock` should change only the `pesu-auth` version entry. If it rewrote other packages, you ran it
with an upgrade flag or a changed dependency; inspect `git diff uv.lock` and keep dependency changes
in their own commit with their own justification.
