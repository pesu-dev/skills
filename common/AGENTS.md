# pesu-dev: shared conventions

These rules apply to every pesu-dev repository. A project's own `AGENTS.md` adds to them and wins
where the two disagree.

## Contributing

- Pull requests come **from a fork**, from a branch that is not `main`, and **target `dev`**. `main`
  is advanced only by the production workflow. `source.yaml` fails any PR that breaks this.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/): `feat:`,
  `fix:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`.
- Fill in the repository's `.github/PULL_REQUEST_TEMPLATE.md` rather than writing a free-form
  description.
- Reviewers come from `.github/CODEOWNERS`. Do not edit it to route around a review.

## Tooling

- Python projects use [uv](https://docs.astral.sh/uv/). Install with `uv sync` and run everything
  through `uv run`. Never `pip install` into the environment by hand.
- Linting and formatting is ruff, run through pre-commit: `uv run pre-commit run --all-files`
  runs the same hooks CI does. Run it before declaring a change done.
- Do not commit `.env`. Every variable a service needs is documented in `.env.example`; update it
  when you add one.

## Agent skills

`AGENTS.md` and the skills in `.agents/skills` come from the
[pesu-dev/skills](https://github.com/pesu-dev/skills) repository, mounted as a git submodule at
`.agents/pesudev-skills`. Do not edit `AGENTS.md`, `.agents/` or `.claude/skills` in a project
repository: open a pull request against pesu-dev/skills instead.
