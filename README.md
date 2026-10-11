# the-cookbook

A personal cookbook for [`the-cook`](https://github.com/lmarqs/the-cook): the ingredients, stations and recipes its
author uses in their own repositories, published by tag so any repository can pin them. It is one cookbook among any
number, not an official or a default one; `the-cook` names no cookbook and ships none of what is here.

## What a cookbook holds

`the-cook` enforces the software development lifecycle a repository declares and knows no program itself. A cookbook
describes the programs and the ways of working, in three units, and one rule separates them: facts about programs are
pinned and rendered; practices are copied and become the repository's own.

| Unit | What it is | How a repository holds it |
| --- | --- | --- |
| [Ingredient](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md) | the facts about one program: the contract it fills, the files rendered for it, its floor and its checks | pinned by source, version and sha256; acts on every render and every launch |
| [Station](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md) | what one way of working needs in place: the ingredients to pin, the workers and their roles, entries, templates and the recipes to add | applied once, in one reviewed change; what it wrote is the repository's own |
| [Recipe](https://github.com/lmarqs/the-cook/blob/main/docs/features/recipes.md) | one kind of change, made many times: inputs, requirements, steps and the lanes it requires | copied into `.cook/recipes/<name>/` with its origin recorded, and edited like any other file |

## Contents

| Kind | Name | What it describes | Page |
| --- | --- | --- | --- |
| Ingredient | `git` | git as the version-control program | [git](docs/ingredients/git.md) |
| Ingredient | `claude` | Claude Code as a worker | [claude](docs/ingredients/claude.md) |
| Ingredient | `codex` | the Codex CLI as a worker | [codex](docs/ingredients/codex.md) |
| Ingredient | `ralph-tui` | ralph-tui as the loop runner | [ralph-tui](docs/ingredients/ralph-tui.md) |
| Ingredient | `beads` | Beads (`bd`) as the task tracker | [beads](docs/ingredients/beads.md) |
| Ingredient | `mise` | mise as the task runner | [mise](docs/ingredients/mise.md) |
| Ingredient | `skills` | the `skills` command as the skills installer | [skills](docs/ingredients/skills.md) |
| Station | `agentic-coding` | an interactive and an unattended coding agent on git, a loop runner and a tracker | [agentic-coding](docs/stations/agentic-coding.md) |
| Station | `go`, `node`, `terraform`, `python` | the commands, lanes and post-edit hooks of each stack | [go](docs/stations/go.md), [node](docs/stations/node.md), [terraform](docs/stations/terraform.md), [python](docs/stations/python.md) |
| Recipe | `implement-task` | one tracker task made into one verified change by the unattended worker | [implement-task](docs/recipes/implement-task.md) |

How several of them combine for one goal:

- [Implement a Beads epic unattended, one verified branch per task](docs/use-cases/unattended-beads-epic.md)
- [Run stories that build on one another on one branch](docs/use-cases/dependent-stories-one-branch.md)
- [Work on one Beads task in its own workspace](docs/use-cases/one-task-by-hand.md)
- [Let a Claude Code session run an epic and prepare its delivery](docs/use-cases/coordinated-run.md)

## Using it

Apply a way of working and a stack to a repository, in one reviewed diff:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --station node
```

Or, in a repository that already uses `the-cook`:

```sh
./cook station apply agentic-coding node --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook ingredient pin mise --cookbook github:lmarqs/the-cookbook@v0.1.0
```

`agentic-coding` copies the `implement-task` recipe into `.cook/recipes/implement-task/` as it applies; from then on
the copy is the repository's own. A repository that applies no station takes a recipe with
`./cook recipe add <name> --cookbook github:lmarqs/the-cookbook@v0.1.0`.

Read what you pin before you approve it: `./cook ingredient show <name> --cookbook github:lmarqs/the-cookbook@v0.1.0`
prints an ingredient's floor and renders, and the station's diff shows every entry it writes. That review is the trust
decision for everything the pinned units run.

## Layout

```text
ingredients/<name>/ingredient.yaml     an ingredient, with files/ for its render templates
stations/<name>/station.yaml           a station, with files/ for its templates
recipes/<name>/recipe.yaml             a recipe, with files/ for its template steps
docs/ingredients/<name>.md             the page for each unit, by kind
docs/stations/<name>.md
docs/recipes/<name>.md
docs/use-cases/<slug>.md               how several units combine for one goal
```

## Tags and hashes

Every tag publishes one archive per unit, named by its kind (`ingredient-<name>.tar.gz`, `station-<name>.tar.gz`,
`recipe-<name>.tar.gz`), and one `checksums.txt` with the sha256 of each. Every ingredient and station carries the tag
as its `version`; a recipe carries none, and a repository's copy records the tag in its `origin`. An archive published
under a tag is never replaced: a fix is a new tag.

A station pins the ingredients and recipes of the same tag by sha256, and an archive's hash exists only once the tag's
archives are built. In this source tree, every such `sha256` is 64 zeros; the release build writes the real hash into
each station archive, after the ingredient and recipe archives and before `checksums.txt`. The first tag is `v0.1.0`,
and no tag is published yet.

## Versions

Each ingredient's `checks.version` names the range of program versions it is written for, and its page states the
version its facts were checked against. `./cook doctor` reports a program whose version is outside that range.
