# Let a Claude Code session run an epic and prepare its delivery

## Who it is for

A developer who hands the whole loop to a coordinating Claude Code session: start ralph-tui over an epic, watch it,
report the tasks that stopped, and arrange the finished task branches into pull requests, with the developer
confirming each delivery.

## What it uses

| Kind | Unit | Role here |
| --- | --- | --- |
| Ingredient | [`claude`](../ingredients/claude.md) | the coordinating worker, declared `interactive` |
| Ingredient | [`codex`](../ingredients/codex.md) | the unattended worker each task's run starts |
| Ingredient | [`ralph-tui`](../ingredients/ralph-tui.md) | the loop runner the session starts |
| Ingredient | [`beads`](../ingredients/beads.md) | the tracker |
| Ingredient | [`git`](../ingredients/git.md) | the task branches the session arranges |
| Station | [`agentic-coding`](../stations/agentic-coding.md) | the workers, the denies, the tracker actions and ralph-tui's configuration |
| Recipe | [`implement-task`](../recipes/implement-task.md) | each task's run |

## Set up

The station applied and committed, the epic in Beads, as in
[Implement a Beads epic unattended](unattended-beads-epic.md#set-up), without per-task `deliver` actions: delivery is
the session's judgement. Allow the session the commands of the run in `.cook/policy.yaml`, with the reason above them:

```yaml
shared:
  allow:
    # The coordinating session starts and watches the loop runner; each task runs under the-cook make.
    - [ralph-tui, [run, status]]
    # Reads of the tracker.
    - [bd, [show, list, ready]]
```

The seeded `ask` on `[the-cook, lifecycle, run, deliver]` stays, so every delivery waits for the developer
([the seeded entry](https://github.com/lmarqs/the-cook/blob/main/docs/features/policy.md#the-seeded-entry)). Run
`./cook render`, review the diff and commit.

## How a run goes

The developer starts the session in the primary checkout and states the boundary of the work:

```sh
claude --permission-mode auto
```

> Run the Beads epic acme-api-a10 with `ralph-tui run --epic acme-api-a10 --headless`. When it ends, list the stopped
> tasks with their reasons. Then group the finished task branches into pull requests by area, and deliver each group
> from a workspace of its own. Do not push anything else.

The session, guided by the shipped skill of `the-cook`:

1. starts ralph-tui headless and follows it with `ralph-tui status --json`;
2. reads each stopped task with `./cook make --status task-<id>` and `bd show <id>`, and reports it;
3. for each pull request, creates a workspace from `origin/main`, cherry-picks the task commits that belong in it,
   rewords their messages while keeping each verification summary, and runs `./cook verify`;
4. runs `./cook lifecycle run deliver` in each of those workspaces, which asks the developer first.

Each task's run is unattended and under the attempt contract whatever the session does: the session starts ralph-tui,
and ralph-tui starts `the-cook make`. After the merges, `./cook workspace remove <name>` fires `landed` for each
workspace.

## Limits

- The session is an interactive worker in its program's unattended mode, under the shared policy, not under the attempt
  contract: what it may run is what the policy allows, in the mode its ingredient's page describes.
- Permitting `ralph-tui run` permits every argument after it, `--agent` included, which can start a program outside
  `the-cook`
  ([ralph-tui](../ingredients/ralph-tui.md#an-agent-named-on-the-command-line-can-start-its-program-directly)).
  Read the commands in the session's transcript.
- Delivery waits at the seeded `ask` for every workspace. Moving the entry to `allow` lets the session deliver unasked,
  which permits everything the `deliver` actions run.
- The session runs with the developer's credentials: start it from a shell without production or administrator
  credentials.
