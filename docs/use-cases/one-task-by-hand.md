# Work on one Beads task in its own workspace

## Who it is for

A developer who picks up one bead and wants the tracker to follow the work: the bead claimed when the work starts,
closed only once its checks pass, and marked with the reason when the work stops. They work on it with Claude Code, or
hand it to Codex and stay at the terminal.

## What it uses

| Kind | Unit | Role here |
| --- | --- | --- |
| Ingredient | [`git`](../ingredients/git.md) | the workspace and its branch |
| Ingredient | [`claude`](../ingredients/claude.md) | the interactive worker the developer works with |
| Ingredient | [`codex`](../ingredients/codex.md) | the unattended worker of the recipe's `attempt` step |
| Ingredient | [`beads`](../ingredients/beads.md) | the tracker |
| Station | [`agentic-coding`](../stations/agentic-coding.md) | the workers and the tracker actions `claim-task`, `close-task` and `block-task` |
| Recipe | [`implement-task`](../recipes/implement-task.md) | the second way: the task made by Codex in one run |

## Set up

The station applied and committed, as in
[Implement a Beads epic unattended](unattended-beads-epic.md#set-up), and the stack's lanes from a stack station.

## How a run goes

With Claude Code, in a workspace bound to the bead:

```sh
./cook workspace create invoice-rounding --task acme-api-7f2   # claim-task marks acme-api-7f2 in progress
cd .work/workspaces/invoice-rounding
claude                                                         # implement and test with Claude Code
./cook verify --lane unit=invoice
git add -A && git commit -m "fix(invoice): round totals half-even"
./cook lifecycle run completed                                 # close-task closes acme-api-7f2
./cook lifecycle run deliver
```

`./cook lifecycle run completed` is refused without a valid receipt, so the bead closes only after `./cook verify`
passed on the final content. Every event fired in the workspace carries the bead's id in `COOK_TASK`, and the bead's
assignee is the workspace's actor, `<user>:invoice-rounding:<random>`.

With Codex, in one recipe run the developer drives:

```sh
bd show acme-api-7f2 | ./cook make implement-task --task acme-api-7f2 --commit
```

The run creates `task-acme-api-7f2`, claims the bead, starts Codex on the bead's text, verifies, replays, commits and
closes the bead, then runs any automatic `deliver` action. When it stops, `block-task` marks the bead and the run
prints where it stopped; the developer fixes it in the workspace and resumes:

```sh
./cook make --status task-acme-api-7f2
cd .work/workspaces/task-acme-api-7f2
./cook make --continue
```

## Limits

- One task per workspace. A change made once that no tracker holds needs no `--task`, and no tracker action runs for
  it.
- The recipe's worker is the unattended one, Codex, even when a person drives the run; the person's own Claude Code
  session is the first way.
- A bead id means something only where its database is: two clones with separate databases have separate beads.
- The tracker actions run `bd` with the credentials of whoever fires the event, and write the primary checkout's
  database.
