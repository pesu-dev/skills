---
name: open-pull-request
description: Turn a finished pesu-auth branch into a pull request - Conventional Commits with explanatory bodies, rebase on origin/dev, final checks, push the branch to pesu-dev/auth, and gh pr create to dev with the PR template filled in truthfully. Use as the last step of any change. Never merges.
---

# Opening a pull request

Policy: the PR comes **from a branch of `pesu-dev/auth` itself**, not a fork
(`.github/CONTRIBUTING.md`), that branch is **not `dev` or `main`**, and the PR targets **`dev`**.
CI's *PR source checker* enforces all three.

## 1. Preconditions

- Definition of done in `.agents/pesudev-skills/auth/docs/autonomous-workflow.md` is met.
- Version bumped once (`bump-version`).
- `git status` is clean; `.env`, benchmark results and scratch files are not staged (`.gitignore`
  covers `.env`, `*.csv`, `*.png`).

## 2. Commits

Conventional Commits, matching `git log` in the repo:

```
<type>[!]: <what changed, imperative, lowercase, no period>

<Why the change was needed: the problem, who it affects.>

<How it works and why this approach, including alternatives rejected and library behaviour relied
on. Mention anything a reviewer would otherwise have to rediscover.>

<Verification: tests added, counts, coverage, live tests run or not.>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`; `!` after the type (and a
`BREAKING CHANGE:` paragraph) for incompatible changes. Keep the version bump in its own
`chore: bump version to X.Y.Z` commit. Wrap bodies at about 100 columns. Add any attribution
trailers your tool or the human requires.

Tidy history before pushing if it contains fixup noise:
`git rebase -i` is interactive; instead use `git reset --soft origin/dev` and recommit in logical
units, or leave the history as is (PRs are squash-merged).

## 3. Rebase and final checks

```bash
git fetch origin
git rebase origin/dev
uv sync --all-groups
uv run pre-commit run --all-files
```

Re-check the version against the new base (`bump-version`) if `dev` moved.

## 4. Push the branch

```bash
git remote -v                                # origin must be pesu-dev/auth
git branch --show-current                    # never dev or main
git push -u origin HEAD
```

If `origin` is a fork, the clone predates the move away from forks: load `setup-environment`,
which re-points the remotes, then push. Push only to a branch you created; if the push is rejected
because the branch already exists on `origin`, pick another name rather than forcing it.

## 5. Create the PR

Title: the Conventional Commit line of the change (e.g. `feat: add apiVersion to /health`). Use
`feat!:`/`fix!:` for a breaking change, and link the issue in the body, as the template's comment
says.

Body: `.github/PULL_REQUEST_TEMPLATE.md`, filled in truthfully:

- **Description:** purpose, problem, context; `Fixes: #<n>` / `Related: #<n>`; delete the HTML comment.
- **Type of Change:** tick only what applies (breaking changes must be ticked and explained).
- **How Has This Been Tested?:** tick the suites you actually ran; test configuration (OS,
  `Python 3.14 via uv`, Docker build tested or not). State plainly if live tests were not run.
- **Checklist:** tick only items actually done; leave others unticked.
- **Affected API Behaviour:** tick the files touched.
- **Screenshots / API Demos:** real `curl` output for behaviour changes (redact credentials and
  personal data). Mandatory for breaking changes.
- **Additional Notes:** limitations, follow-ups, anything not verified.

```bash
gh pr create --repo pesu-dev/auth --base dev \
  --head <branch> \
  --title "feat: ..." --body-file /path/outside/repo/pr-body.md
```

End the body with any attribution line your tool or the human requires.

## 6. After opening

Report the PR URL. Watch the checks (`gh pr checks <n> --repo pesu-dev/auth --watch`) and load
`fix-ci` for any failure. CI runs the live tests on the PR; say in the PR whether they passed. Do
not merge, request merges, or ping maintainers repeatedly.
