# Implement a Beads epic unattended, one verified branch per task

## Who it is for

A team that keeps its backlog in Beads and breaks an epic into small tasks that do not depend on one another. It wants
Codex to implement them while nobody watches, and to come back to one branch per task whose checks passed, each one a
commit ready for review, and every task that did not pass stopped where it stopped, with its reason on the bead.

## What it uses

| Kind | Unit | Role here |
| --- | --- | --- |
| Ingredient | [`git`](../ingredients/git.md) | version control: workspaces, bases, commits |
| Ingredient | [`claude`](../ingredients/claude.md) | the interactive worker, for the person who prepares the epic and picks up stopped tasks |
| Ingredient | [`codex`](../ingredients/codex.md) | the unattended worker that implements each task |
| Ingredient | [`ralph-tui`](../ingredients/ralph-tui.md) | the loop runner that takes each ready bead |
| Ingredient | [`beads`](../ingredients/beads.md) | the tracker |
| Station | [`agentic-coding`](../stations/agentic-coding.md) | pins the five, declares the workers, writes the denies, the tracker actions and ralph-tui's configuration |
| Station | [`node`](../stations/node.md) | the commands and lanes of a Node API; another stack station for another stack |
| Recipe | [`implement-task`](../recipes/implement-task.md) | one task, one workspace, one verified commit |

## Set up

Once per repository, from the root of the primary checkout of a Node API, `acme-api`:

```sh
./cook station show agentic-coding --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply agentic-coding node --cookbook github:lmarqs/the-cookbook@v0.1.0
bd init --skip-hooks --skip-agents        # only when the clone has no Beads database
git add -A && git commit -m "chore: run Beads epics through the-cook"
```

Each task delivers on its own when the repository declares `deliver` actions with scope `task` and `auto: true`. For a
repository on GitHub, add them to `.cook/hooks.yaml` and commit:

```yaml
lifecycle:
  deliver:
    - name: push
      run: git push -u origin "$COOK_BRANCH"
      auto: true
      scope: [task]
    - name: open-pr
      run: >-
        gh pr view "$COOK_BRANCH" >/dev/null 2>&1 ||
        gh pr create --draft --fill --base "$COOK_DEFAULT_BRANCH" --head "$COOK_BRANCH"
      auto: true
      scope: [task]
```

Without them, the task branches stay local for a person to deliver after judgement.

Once per machine: `./cook fetch`, and `ralph-tui config show --sources` to see whether a per-user configuration sets a
key the project's file leaves unset.

Create the epic and its tasks. The first `bd create` prints only the epic's id (`--silent`), here `acme-api-a10`; each
task names it with `--parent` and gets a child id, `acme-api-a10.1` and `acme-api-a10.2`:

```sh
bd create --type epic --title "Invoice rounding" --no-inherit-labels --body-file epic.md --silent
bd create --parent acme-api-a10 --title "US-001: Round invoice totals half-even" --no-inherit-labels --body-file us-001.md --silent
bd create --parent acme-api-a10 --title "US-002: Round credit notes half-even" --no-inherit-labels --body-file us-002.md --silent
```

Each body file holds the whole task, because the task's text is all the worker gets, and ends with the criteria:

```markdown
Round each invoice total half-even to two decimals, once, in `InvoiceTotals`. Amounts stay unrounded before that.

## Acceptance Criteria
- [ ] `InvoiceTotals` rounds half-even to two decimals
- [ ] A unit test covers 0.125, 0.135 and a negative amount
```

## How a run goes

```sh
ralph-tui run --epic acme-api-a10 --headless
```

ralph-tui takes the epic's ready beads one at a time, in the primary checkout, and for each one runs
`./cook make implement-task --task <id> --unattended --commit --withhold '<promise>COMPLETE</promise>'` with the bead's
text on stdin:

| Stage | For `acme-api-a10.1` |
| --- | --- |
| Start | the workspace `task-acme-api-a10.1` on `work/task-acme-api-a10.1` from `main`; `claim-task` marks the bead in progress |
| Implement | Codex works on the task in the workspace, under the attempt contract, and runs `./cook verify` with the lanes its change needs |
| Verify | the run runs those lanes itself, adds those the lane rules demand, and writes the receipt |
| Complete | the replay from the base; the commit; `completed`, where `close-task` closes the bead; `deliver`, where `push` and `open-pr` run |
| A stop | `blocked`, where `block-task` marks the bead blocked with the reason; the workspace stays; ralph-tui skips the task |

The run ends when no bead of the epic is ready, or after ten iterations of one task each, the budget
`maxIterations` sets. Then:

```sh
git branch --list 'work/task-*'                       # one branch per task that passed
bd list --parent acme-api-a10 --status blocked        # the tasks that stopped, with the reason in their notes
./cook make --status task-acme-api-a10.2              # where a stopped run stopped: the stage, the step, the reason
```

A person picks up a stopped task in its workspace, often with Claude Code: fixes what stopped it, or runs
`./cook verify` with the lane the message names, then resumes the run, which verifies, commits and closes the bead:

```sh
cd .work/workspaces/task-acme-api-a10.2
./cook make --continue
```

Without per-task `deliver` actions, deliver after judgement: choose the task commits that belong together, put them on
one branch in a workspace of their own, verify, and run `./cook lifecycle run deliver` there. Once a task's change is
merged, `./cook workspace remove task-acme-api-a10.1` fires `landed` and removes its workspace.

## Limits

- Every task starts from the default branch, so a task that needs another one's change runs without it until that
  change is merged. For stories that build on one another, see
  [Run stories that build on one another on one branch](dependent-stories-one-branch.md).
- Tasks run one after another. The station keeps ralph-tui out of parallel mode, because each run makes its own
  workspace.
- A stopped task waits for a person: neither ralph-tui nor the run retries it.
- It suits work whose checks run on the machine. A repository whose checks need credentials and judgement an
  unattended worker must not hold, such as `terraform plan` against real infrastructure, keeps its changes attended.
- The worker gets only the bead's text and the repository; a task that needs a decision only a person can make stops,
  as the directive tells the worker.
