# Run stories that build on one another on one branch

## Who it is for

A developer whose stories each build on the one before, such as a CLI command and then its flags, who wants them
implemented unattended in order, each one a verified commit on top of the last, on one branch that becomes one pull
request or is arranged into several after judgement.

## What it uses

| Kind | Unit | Role here |
| --- | --- | --- |
| Ingredient | [`git`](../ingredients/git.md) | version control: the run's workspace, each attempt's base, the commits |
| Ingredient | [`codex`](../ingredients/codex.md) | the unattended worker of every attempt |
| Ingredient | [`ralph-tui`](../ingredients/ralph-tui.md) | the loop runner, here with an agent entry that runs one attempt per task |
| Ingredient | [`beads`](../ingredients/beads.md) | the tracker, whose dependencies order the stories |
| Station | [`agentic-coding`](../stations/agentic-coding.md) | the pins, the workers and the unattended denies |
| Station | [`go`](../stations/go.md) | the commands and lanes of a Go CLI |

No recipe: each story runs as one
[`the-cook attempt`](https://github.com/lmarqs/the-cook/blob/main/docs/features/attempts.md) in the run's one
workspace, so each attempt's base is the previous story's commit.

## Set up

With the station applied, add a second agent entry to `.ralph-tui/config.toml`, the repository's own file:

```toml
[[agents]]
name = "attempt"
plugin = "codex"
command = "bash"
defaultFlags = [
  "-c",
  '''
exec ./cook attempt codex --commit --withhold '<promise>COMPLETE</promise>' <<EOF
$(sed '1{/^COOK_TASK_ID=/d;}')

When your change is done and ./cook verify passed, end your reply with <promise>COMPLETE</promise>.
EOF
''',
  "attempt",
]

[agents.rateLimitHandling]
enabled = false
```

ralph-tui renders the prompt from the station's `.ralph-tui/templates/beads.hbs` for this entry too. The command drops
its `COOK_TASK_ID=` line, which an attempt does not need, and asks the worker for ralph-tui's tag, which that template
leaves out because `the-cook make` prints it. `the-cook attempt` prints the worker's output as it is for a verified
attempt, the tag included, and removes the tag from a failed one, and ralph-tui closes the bead once it reads the tag.
Every argument ralph-tui's `codex` plugin appends is a positional parameter the script never reads.

Declare delivery for the whole run, with scope `run`, which only a caller fires:

```yaml
lifecycle:
  deliver:
    - name: push-run
      run: git push -u origin "$COOK_BRANCH"
      scope: [run]
    - name: open-run-pr
      run: >-
        gh pr create --draft --base "$COOK_DEFAULT_BRANCH" --head "$COOK_BRANCH"
        --title "Run $COOK_WORKSPACE_NAME"
        --body "One commit per story; each commit body carries its verification summary."
      scope: [run]
```

Commit both before the run: the run's workspace starts from committed code, and `the-cook attempt` reads the rules
from its base. Then create the stories, each depending on the one before, on the story and not on the epic:

```sh
bd create --type epic --title "Export to CSV" --no-inherit-labels --body-file epic.md --silent      # acme-cli-b20
bd create --parent acme-cli-b20 --title "US-001: Add the export command" --no-inherit-labels --body-file us-001.md --silent
bd create --parent acme-cli-b20 --title "US-002: Add --columns to export" --no-inherit-labels --body-file us-002.md --silent
bd dep add acme-cli-b20.2 acme-cli-b20.1
```

## How a run goes

```sh
./cook workspace create run-export
beads="$PWD/.beads"
cd .work/workspaces/run-export
BEADS_DIR="$beads" ralph-tui run --agent attempt --epic acme-cli-b20 --headless
./cook lifecycle run completed --scope run    # every story finished
./cook lifecycle run blocked --scope run      # instead, when stories were left open
```

The run's workspace is bound to no task, so the attempts fire no event and the station's tracker actions do not run:
ralph-tui's Beads plugin marks each bead in progress and closes it when it reads the tag. `BEADS_DIR` points ralph-tui's
`bd` at the primary checkout's database from inside the workspace.

| Story | What happens |
| --- | --- |
| `acme-cli-b20.1` | the only ready bead; Codex implements it; the replay passes and `--commit` makes one commit on `work/run-export`; ralph-tui closes the bead |
| `acme-cli-b20.2` | ready once `.1` closed; its attempt starts from `.1`'s commit; the same steps |
| A failed attempt | the checkout returns to its base, the changes are kept as `last-attempt.patch` in the workspace's state directory, the tag is removed; ralph-tui skips the bead, and the beads that depend on it never become ready |

The branch `work/run-export` then holds one commit per story whose checks passed. Deliver it as one pull request:

```sh
./cook lifecycle run deliver --scope run      # push-run, then open-run-pr
```

Or deliver after judgement: create a workspace from `origin/main`, cherry-pick the story commits that belong in one
pull request, reword their messages while keeping the verification summary in each body, run `./cook verify`, and run
`./cook lifecycle run deliver` there. After the merge, `./cook workspace remove run-export` fires `landed` and removes
the run's workspace.

A JSON list of user stories works the same way: keep the file in the primary checkout's ignored state directory, such
as `.work/prds/export-csv.json`, and pass `--prd <path>` instead of `--epic`. ralph-tui marks each story done by
setting its `passes` to `true` in that file, so nobody edits the file by hand during the run.

## Limits

- Stories run one after another. ralph-tui's parallel mode runs each story in a worktree beside the repository, outside
  the primary checkout, where a worker that trusts project settings per folder may not apply the repository's rules;
  this use case does not cover it.
- A story that changes `.cook/` or a file standing context names fails its attempt, by the tamper check: make such
  changes interactively and commit them before the run.
- The beads close when their attempt passes, before the branch is reviewed; a story a reviewer rejects is reopened by
  hand.
- One failed story ends the run for every story after it.
