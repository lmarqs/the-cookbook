# skills

The `skills` ingredient describes the `skills` CLI, the installer that puts shared agent skills into `.agents/skills/`
and records them in `skills-lock.json`. Pinned, it lets `doctor` check the CLI's version on each machine that runs it;
installing, updating and removing skills stay the CLI's own commands, run by a person or a worker.

## What it describes

- **Program.** The [`skills` CLI](https://www.npmjs.com/package/skills), the npm package `skills`, its 1.7 releases,
  run through `npx`. Checked against 1.7.2. Its package declares Node `>=22.20.0`.
- **Contract.** None. `the-cook` never runs the CLI of its own accord.
- **Detect.** `skills-lock.json`.

What the CLI does, read from its package files:

- `npx skills add <source>` copies each skill into `.agents/skills/<name>/` under the current directory, and links it
  into the skills directory of each agent it installs for, such as `.claude/skills/<name>` for Claude Code, with a
  relative symbolic link; `--copy` copies instead of linking. An agent whose skills directory is `.agents/skills`
  itself, such as Codex, gets no link. With `-g` it installs under the home directory instead.
- It records each skill it installs in a project in `skills-lock.json`, in the current directory: its source, the
  source's type and a sha256 of the skill's files. `npx skills experimental_install` restores the skills the file
  lists.
- When an agent's skills directory is already a link to `.agents/skills`, the link step finds the skill in place and
  changes nothing. When it is a real directory, the CLI's links sit beside the one link a worker ingredient renders
  there for the shipped skill, such as `.claude/skills/the-cook`.

`the-cook` contains no code for the CLI and never reads `skills-lock.json`. The shipped skill,
`.agents/skills/the-cook/`, is in no lock file, so `npx skills update`, which updates only what the lock file lists,
never touches it; a removal can ([Pitfalls](#removing-skills-with-the-cli-removes-the-shipped-skill)). A
[`context`](https://github.com/lmarqs/the-cook/blob/main/docs/features/context.md) entry names an installed skill by its
directory under `.agents/skills/`, and `doctor` reports one that does not exist. A skill reaches a session as it is
committed, and an attempt reads it from its base commit, so the commit that installs a skill is what a reviewer approves
([shared skills](https://github.com/lmarqs/the-cook/blob/main/docs/features/skills.md#shared-skills)). Because the
commit carries the skills, a clone on a machine without Node, such as one that only builds a Terraform or Go repository,
has every skill without running the CLI.

No station of this cookbook lists the ingredient; a repository pins it on its own with `the-cook ingredient pin`.

## Checks

| Check | Value |
| --- | --- |
| `version` | `npx --no skills --version`, which prints `1.7.2`; the range is `>=1.7 <1.8` |
| `detect` | `skills-lock.json` |

`--no` makes npx refuse to install a package it does not find. Without it, npx installs a missing package on its own
whenever standard input is not a terminal, and the check would report the version that download fetched. On a machine
where npx finds no copy of the CLI, the check fails and `doctor` reports it.

## Pitfalls

### Removing skills with the CLI removes the shipped skill

- **Symptom.** After `npx skills remove --all`, or a removal that selects `the-cook`, `.agents/skills/the-cook/` and
  its links are gone, workers do not list the skill, and `./cook render --check` reports drift.
- **Why.** The CLI finds installed skills by scanning `.agents/skills/` and every agent's skills directory for
  directories that hold a `SKILL.md`, whatever `skills-lock.json` lists, and `--all` removes each one it finds, with
  its links.
- **What the ingredient does.** Nothing: it renders nothing. `./cook render` writes the shipped skill and its links
  back. Remove shared skills by name.

### Inside a coding agent's session, the CLI does not ask

- **Symptom.** A worker runs `npx skills add <source>` or `npx skills remove <name>`, and `.agents/skills/` changes with
  no confirmation.
- **Why.** The CLI detects that it runs inside a coding agent and answers its own prompts with yes.
- **What the ingredient does.** Nothing; the policy decides. A skill is instructions workers follow, so deny
  `[npx, skills]` in `unattended.deny`, with `[skills]` where a global install puts the CLI on `PATH`, or put them under
  `ask`, and review every change under `.agents/skills/` like code.

### The CLI writes relative to the current directory

- **Symptom.** `.agents/skills/` and `skills-lock.json` appear in a subdirectory, and no worker finds the skills.
- **Why.** The CLI takes the directory it runs in as the project, not the root of the repository.
- **What the ingredient does.** Nothing. Run the CLI from the root of the checkout.

### Installing a skill replaces one of the same name

- **Symptom.** After `npx skills add <source>`, a skill the repository wrote is gone, replaced by the source's skill of
  the same name.
- **Why.** The CLI deletes `.agents/skills/<name>/` before it copies the new skill there.
- **What the ingredient does.** Nothing. List what a source publishes first, with `npx skills add <source> --list`, and
  read the diff of `.agents/skills/` before committing. A source's skill named `the-cook` collides with the shipped
  skill, which `./cook render` writes back over it.

### The CLI reports installs to skills.sh

- **Symptom.** `npx skills add` sends the source and the names of the skills it installs to skills.sh.
- **Why.** The CLI sends an install event, and asks skills.sh for audit data about the skills; for a source on GitHub,
  only when the repository is public.
- **What the ingredient does.** Nothing. Set `DO_NOT_TRACK=1` or `DISABLE_TELEMETRY=1` in the environment that runs
  the CLI to turn both off.

## Examples

### Install shared skills and commit them

```sh
npx skills add acme/agent-skills
git status                          # the skills, the lock file, and the links in each agent's directory
git add .agents/skills skills-lock.json
git commit -m "chore: install shared agent skills"
```

Add the links the CLI made in each agent's own directory, such as `.claude/skills/`, to the same commit, so every
clone and every worker has them.

### Pin it

```sh
./cook ingredient pin skills --cookbook github:lmarqs/the-cookbook@v0.1.0
```

```yaml
# .cook/config.yaml, the ingredients section
ingredients:
  - name: skills
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

The pin renders nothing. From then on `doctor` reports a CLI outside 1.7 on the machine that runs it.

### Give every session an installed skill

```yaml
# .cook/config.yaml, the context section
context:
  always:
    - skill: coding-guidelines
```

When `.agents/skills/coding-guidelines/` does not exist, `doctor` reports it and nothing is injected for it: install it
with the CLI and commit it, or fix the name.

### Bring an organization's conventions through a skill

An organization publishes a skill that describes its choices, and a coding agent following it writes them into each
repository's own files, reviewed as a diff:

```sh
npx skills add acme/agent-skills
claude "apply the acme agent conventions to this repository"
./cook render
git diff .cook/ AGENTS.md
```

The entries then live in the repository's `.cook/policy.yaml`, `.cook/hooks.yaml` and profile, where its people and
workers read them, and every machine renders the same bytes without fetching anything. Review the `lifecycle`
actions such a change adds to `.cook/hooks.yaml` like code: they run outside every worker's permissions.
