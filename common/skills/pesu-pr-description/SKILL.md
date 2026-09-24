---
name: pesu-pr-description
description: Write a pull request title and description for a pesu-dev repository by filling in the repo's PULL_REQUEST_TEMPLATE.md from the current branch's changes. Use when the user asks to open, draft or describe a PR in any pesu-dev project.
---

# Writing a pesu-dev pull request

1. Find the base. pesu-dev PRs target `dev`, so diff against it:
   `git fetch upstream dev 2>/dev/null || git fetch origin dev`, then
   `git log --oneline <remote>/dev..HEAD` and `git diff <remote>/dev...HEAD --stat`.
1. Read `.github/PULL_REQUEST_TEMPLATE.md`. It is the structure of the description: keep every
   heading, tick only the boxes that are true, and delete HTML comments the template tells you to
   delete. Never invent a section the template does not have.
1. Title: a Conventional Commit line (`feat: ...`, `fix: ...`) under 72 characters that says what
   changed for a user of the project, not which files moved.
1. Description: say what changed and why in two or three sentences, then fill in the template's
   testing section with the commands that were actually run. If something was not run, leave its
   box unticked rather than claiming it.
1. Before opening the PR, check the branch is on a fork and is not `main`; the org's `source.yaml`
   rejects anything else. Open it with `gh pr create --base dev --repo pesu-dev/<project>`.
