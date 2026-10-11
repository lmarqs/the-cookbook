# implement-task

The `implement-task` recipe takes one task of a tracker and makes it one verified change: a workspace of its own bound
to the task, the repository's unattended worker on the task's text, the lanes the change needs, and, with `--commit`,
one commit on a branch of its own. A loop runner names it as its command for each task, and a person runs it for one
task.

```yaml
schema: 1
name: implement-task
description: Implement one task of the tracker in a workspace of its own
requires:
  contracts: [vcs, worker]
  roles: [unattended]
workspace: "task-{task}"
steps:
  - attempt: {}
```

## The change it makes

Whatever the task asks for, made by the worker in the workspace `task-<id>`, on the branch `work/task-<id>` from the
default branch. The run proves the change with the lanes it selects, replays them from the base, and with `--commit`
records it as one commit whose body carries the recipe and the verification summary. The task's tracker actions run at
`start`, `completed` and `blocked`, and the repository's `deliver` actions deliver the work, as for any task.

## Inputs

None. The task id comes with `--task <id>`, a flag of `the-cook make`, and fills `{task}` in the workspace name, so a
run without `--task` exits 1 before it creates anything. The task's text comes on stdin: the `attempt` step has no
prompt, so it passes the worker the prompt the caller of `the-cook make` writes on stdin, and a run whose caller writes
none exits 1 before it creates anything.

A task id becomes part of a workspace name, which takes letters, digits, `.`, `-` and `_`. Bead ids, such as
`acme-api-a10.1`, and issue numbers fit.

## Requires

| Key | Value | Why |
| --- | --- | --- |
| `contracts` | `vcs`, `worker` | the run creates a workspace and reads its base through the version-control ingredient, and starts a worker |
| `roles` | `unattended` | the `attempt` step starts the repository's only worker with role `unattended` |

It requires no lane: every repository names its own, and the change's selection with the
[lane rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/verify.md#lane-rules) decides which run
([Verify](#verify)). `./cook recipe add` and `./cook station apply` check these requirements against the profile, and a
repository that declares no unattended worker gets exit 3, naming the role.

## Steps

| Stage | Key | What the run does |
| --- | --- | --- |
| Plan | `requires` | checks the contracts and the role, and that `--task` is given |
| Start | `workspace` | creates `task-<id>` on `work/task-<id>` from the default branch, bound to the task, runs `bootstrap`, fires `start` with scope `task` |
| Implement | `steps` | one `attempt` step: the only unattended worker, on the prompt from stdin, under the attempt contract |
| Verify | none | runs the lanes the worker's `./cook verify` selected, under the lane rules, and writes the receipt |
| Complete | none | replays the receipt from the base; with `--commit` commits; fires `completed`, then `deliver` when the work is concluded |
| Deliver, Landed | none | the repository's own actions, as for any task |

The `attempt` step names no worker, because worker names are each repository's own: it starts the one worker the
profile declares with role `unattended`, and a profile with more than one needs `worker` in the step
([`attempt`](https://github.com/lmarqs/the-cook/blob/main/docs/features/recipes.md#attempt)). The worker's edits stay in
the workspace, and a worker that fails stops the run as blocked with its changes in place.

## Verify

The recipe has no `verify.require`. The worker runs `./cook verify` with the lanes its change needs, as the
[agentic-coding station](../stations/agentic-coding.md#agentsdirectivesunattendedmd)'s directive and the `verify`
block of `AGENTS.md` tell it; that receipt is the selection the run's Verify starts from. The run then runs those lanes
itself, adds every lane the lane rules demand, such as one whose `required_when` matches, and stops at Verify, naming
the lane, when such a lane was neither selected nor skipped. A worker that ran no `./cook verify` on an application
change therefore stops the run at Verify.

A repository whose every task must pass a lane adds it to its copy, with the lane in `requires.lanes`:

```yaml
requires:
  contracts: [vcs, worker]
  roles: [unattended]
  lanes: [unit]
verify: { require: [unit] }
```

## Running it

The recipe runs from the default branch's head: commit `.cook/recipes/implement-task/` there before the first run.

Attended, for one task, with the task's text from the tracker:

```sh
bd show acme-api-7f2 | ./cook make implement-task --task acme-api-7f2 --commit
```

The person is there to act when the run stops: `./cook make --status task-acme-api-7f2` names the stage, the step and
the reason, and `./cook make --continue task-acme-api-7f2` resumes it once the cause is fixed. The worker of the
`attempt` step is still the unattended one: a recipe's `attempt` step always runs under the attempt contract.

With a loop runner, every task is one unattended run:

```sh
./cook make implement-task --task <id> --unattended --commit --withhold '<promise>COMPLETE</promise>'
```

`--unattended` declares that nobody drives the run; `--withhold` takes the loop runner's completion signal, which the
run prints once it completes and removes from everything else it prints. The
[agentic-coding station](../stations/agentic-coding.md#ralph-tuiconfigtoml) writes this command into ralph-tui's
configuration, reading the id from ralph-tui's prompt.

## Adapting it

- **A required lane.** Add `verify.require`, with the lane in `requires.lanes`, as in [Verify](#verify).
- **A prompt of the recipe's own.** An `attempt` step with a `prompt` ignores stdin, so a fixed prompt such as
  `"Implement task {task}."` needs the worker to read the task itself, which a deny on the tracker's command forbids.
  Keep the prompt on stdin unless the task's text is somewhere the worker may read.
- **A step before the worker.** A `run` step, such as a generator the repository's tasks always need, runs before the
  `attempt` step with the rights of whoever started the run.
- **Another unattended worker.** A profile that declares two unattended workers names one in the step:
  `- attempt: { worker: codex }`.
- **A model or an effort.** `- attempt: { model: <id>, effort: <level> }` overrides the worker's for this recipe only.
- **No handoff.** A `handoff` step stops every unattended run as blocked; keep handoffs out of a recipe a loop runner
  runs.

Take a newer version with `./cook recipe update implement-task --cookbook github:lmarqs/the-cookbook@<tag>`, which
keeps these edits wherever the newer version changed other lines.

## Pitfalls

### A second run of the same task exits 1

- **Symptom.** `make` exits 1 naming the workspace or the branch `work/task-<id>`.
- **Why.** An earlier run of the task kept its workspace and branch: a stopped run keeps both for a person, and a
  completed run keeps them until delivery and `./cook workspace remove`.
- **What the recipe does.** Nothing. Continue the earlier run with `./cook make --continue task-<id>`, or remove its
  workspace and delete the branch.

### Tasks never see each other's changes

- **Symptom.** A task that depends on another one's change runs without it.
- **Why.** Every run creates its workspace from the default branch.
- **What the recipe does.** Nothing. Merge a task's change before the tasks that need it run, or run dependent tasks on
  one branch through `the-cook attempt`
  ([Run stories that build on one another on one branch](../use-cases/dependent-stories-one-branch.md)).

### An edit to the recipe does not reach a stopped run

- **Symptom.** A fixed recipe still stops the run that was stopped.
- **Why.** A run reads its recipe once, when it starts, and `make --continue` runs the content its record holds.
- **What the recipe does.** Nothing. Commit the fix on the default branch for the next runs, and continue or abandon
  the stopped one.
