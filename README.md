# pesu-dev/skills

Agent instructions, skills and roles for pesu-dev projects, kept in one repository. Each project
repo mounts this repo as a git submodule, so any coding agent opened in that project (Codex, Claude
Code, Copilot, Cursor, Gemini and others) finds the project's material straight after cloning.
Contributors never clone this repo themselves.

Everything is plain Markdown in agent-neutral formats: `AGENTS.md` for instructions,
[Agent Skills](https://agentskills.io) (`SKILL.md`) for skills, and plain Markdown files for roles.
There are no scripts to run and nothing is generated.

## Layout

```
common/                        shared across pesu-dev
├── AGENTS.md                  org-wide conventions, to copy from into project AGENTS.md files
└── skills/<name>/SKILL.md     shared skills; projects link to the ones they want
<project>/                     one folder per repo, named after it, owned by its maintainers
├── AGENTS.md                  the project's instructions: the entry point agents read
├── skills/
│   ├── <name>/SKILL.md                       the project's own skills
│   └── <name> -> ../../common/skills/<name>  links to common skills it uses
├── docs/<topic>.md            optional: reference material agents read on demand
└── agents/<role>.md           optional: role definitions (planner, reviewer, ...)
.github/                       CODEOWNERS and the two workflows (see Checks)
.pre-commit-config.yaml        the checks CI runs
```

**`auth/` is the reference project.** It has everything a project folder can have: an `AGENTS.md`,
ten reference docs, ten roles and 24 skills that together let an agent take a change from an issue
to an open pull request on its own. Start a new project by reading it and copying the parts that
fit.

## How a project repo uses this

```
<project repo>/
├── .agents/pesudev-skills/   submodule: this repo
├── AGENTS.md           -> .agents/pesudev-skills/<project>/AGENTS.md
├── .agents/skills      -> pesudev-skills/<project>/skills
└── .claude/skills      -> ../.agents/pesudev-skills/<project>/skills
```

`AGENTS.md` and `.agents/skills` are the standard locations every agent reads. `.claude/skills`
points at the same folder and exists only because Claude Code does not read `.agents/skills`.
Docs and roles need no links: `AGENTS.md` and the skills point agents at
`.agents/pesudev-skills/<project>/docs/` and `.agents/pesudev-skills/<project>/agents/` directly.

The links point at folders, so they never change. A new skill, doc or instruction reaches a project
when its submodule is bumped.

## Writing a project folder

### `AGENTS.md`

The one file every agent reads in full, so keep it to a few hundred lines and point to the rest.
`auth/AGENTS.md` shows the shape:

1. What the project is and where it runs.
1. Where the agent material lives (skills, docs, roles) and how short paths resolve.
1. Quick-reference commands: set up, run, test, lint, build.
1. A map of the repository.
1. Rules that always apply, each pointing to the doc with the detail.
1. The contribution workflow (branches, commits, PRs, versioning).
1. What an agent may do on its own and when it must stop.
1. An index of skills, roles and docs, each with "use when".

Only include what an agent cannot work out from the code. Copy what applies from `common/AGENTS.md`
into it: an agent reads a single `AGENTS.md`, so org-wide rules only apply if they are in the
project's file (or it explicitly tells agents to read `.agents/pesudev-skills/common/AGENTS.md`).

### Skills

A skill is a folder with a `SKILL.md`: YAML frontmatter, then the instructions an agent follows once
it loads the skill.

```markdown
---
name: run-tests
description: What the skill does and when to use it. Agents read only this to decide whether to load it, so name the task and the situations that call for it.
---

# Running tests

Numbered steps with the exact commands and file paths, checklists, and how to verify the result.
```

- `name` must match the folder name: lowercase letters, digits and hyphens. Agents skip a skill
  whose `name` or `description` is missing, without any error.
- Write for the task, not the topic: `add-endpoint`, `fix-ci`, `bump-version`. One skill should be
  the orchestrator that runs a whole change end to end and calls the others (`auth`'s
  `implement-feature`).
- Give exact commands and repository paths, the checks that prove the work is done, and what the
  agent must never do.
- Scripts, templates or reference files a skill needs go next to its `SKILL.md`.
- Keep Python code blocks formatted the way the project's ruff would format them. Ruff formats code
  blocks in Markdown, and inside a project repo it sees the submodule, so a badly formatted example
  makes a local `ruff format --check .` fail there.

### Docs

Longer reference material, in `<project>/docs/<topic>.md`: architecture, conventions, testing, API
contract, CI/CD, security, and so on. Agents open a doc when `AGENTS.md` or a skill points them to
it, so every doc should be linked from somewhere. A `known-issues.md` listing things in the project
that are wrong or misleading stops agents from copying them.

### Roles

A role describes one phase of autonomous work: planning, implementing, testing, reviewing,
releasing. It is plain Markdown with only `name` and `description` frontmatter, so it works with
any agent: tools that can start sub-agents run the role as one, and other tools follow the file
themselves for that phase. `AGENTS.md` or a skill says which role to use when.

```markdown
---
name: reviewer
description: What the role does and when to use it. Say whether it is read-only.
---

# Reviewer

## Inputs
## Do
## Output
## Never
## Stop and escalate when
```

### Paths

Inside a project repo, this repo is at `.agents/pesudev-skills/`. Skills, docs and roles refer to each
other and to the project's code by paths from the project repo's root (for example
`.agents/pesudev-skills/auth/docs/testing.md` or `app/pesu.py`), never by relative paths: a skill read
through the `.agents/skills` link would resolve them to the wrong place.

## Common skills

Link a common skill into a project's `skills/` folder to use it; later changes to it then reach the
project automatically:

```bash
ln -s ../../common/skills/pesu-review <project>/skills/pesu-review
```

A newly added common skill is not linked anywhere until a project's maintainers link it. Do not
link a common skill that a project skill already covers (`auth` has its own `review-change`, for
example): agents would have two competing skills for the same task.

## Adding a project

1. Here: create `<project>/` with an `AGENTS.md` and a `skills/` folder containing at least one
   skill or common-skill link (git does not track empty folders), and add the maintainers to
   `.github/CODEOWNERS`.

1. In the project repo, on a branch, add the submodule and the links:

   ```bash
   git submodule add https://github.com/pesu-dev/skills.git .agents/pesudev-skills
   ln -s .agents/pesudev-skills/<project>/AGENTS.md AGENTS.md
   ln -s pesudev-skills/<project>/skills .agents/skills
   mkdir -p .claude && ln -s ../.agents/pesudev-skills/<project>/skills .claude/skills
   git add .gitmodules .agents AGENTS.md .claude/skills
   ```

1. Still in the project repo, add `.github/dependabot.yml`, so updates to this repo arrive as pull
   requests that bump the submodule:

   ```yaml
   version: 2
   updates:
     - package-ecosystem: gitsubmodule
       directory: /
       schedule:
         interval: daily
       target-branch: dev
   ```

   Workflows that only accept pull requests from forks, or that require a version bump, need an
   exception for `dependabot[bot]`.

1. Add the cloning instructions below to the project's README or `CONTRIBUTING.md`.

## Cloning a project

```bash
git clone --recurse-submodules https://github.com/<you>/<project>.git
# already cloned without it:
git submodule update --init
```

Without the submodule, the links point at nothing, so agents see no skills and no instructions.

On Windows, turn on Developer Mode and run `git config --global core.symlinks true` **before**
cloning, or git checks the links out as plain text files.

## Changing anything

Open a pull request here, not in the project repo: everything under `.agents/pesudev-skills` is a
pinned checkout of this repo. Like the pesu-dev projects, pull requests come from a branch of your
fork that is not `main`, and target `main` (this repo has no `dev` branch, since nothing here is
deployed). Once it merges, Dependabot opens a pull request in each project to
bump the submodule. To try an unmerged change, run `git -C .agents/pesudev-skills checkout <branch>`
inside the project.

## Checks

Two workflows run on every pull request:

| Workflow                                                   | Checks                                                                                                                                                                                                                                             |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Enforce PR Origin Policy (`.github/workflows/source.yaml`) | The pull request comes from a fork, not from its `main`, and targets `main`                                                                                                                                                                        |
| Pre-Commit Checks (`.github/workflows/pre-commit.yaml`)    | The hooks in `.pre-commit-config.yaml`: file hygiene, links that point nowhere or were checked out as plain files, Markdown formatting (`mdformat`, keeping YAML frontmatter intact), and ruff formatting of Python code blocks at line length 120 |

Nothing needs installing to contribute: CI runs the hooks, and a failure shows the diff to apply.
To run them locally anyway: `uvx pre-commit run --all-files`.

## Who owns what

`common/` and this README are shared. Each project folder belongs to that project's maintainers
(see `.github/CODEOWNERS`): they decide what its `AGENTS.md` says and which skills, docs and roles
it has.
