---
name: pesu-review
description: Review a change in a pesu-dev repository before it is pushed. Checks it against AGENTS.md, runs the project's checks, and reports concrete problems. Use after finishing a change and before opening a pull request, or when asked to review a branch.
---

# Reviewing a pesu-dev change

1. Read `AGENTS.md` at the repository root. It has the org conventions and the project's own rules.
1. Collect the change: `git diff dev...HEAD` plus `git status` for anything uncommitted.
1. Run `uv run pre-commit run --all-files` and any check `AGENTS.md` lists as required. Report
   failures verbatim.
1. Read the changed code. Look for bugs, missed edge cases, changes that break a contract the
   project documents, and missing tests or documentation updates the project requires.

Report findings as a list, most severe first, each with `file:line`, what is wrong and a concrete
failing scenario. Say plainly if nothing is wrong. Do not edit files while reviewing.
