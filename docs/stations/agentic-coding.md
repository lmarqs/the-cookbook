# agentic-coding

The `agentic-coding` station sets a repository up to run its lifecycle with two coding agents on git: Claude Code in
sessions with a person, and Codex unattended, in a loop run where ralph-tui makes each ready bead of a Beads epic one
run of the [`implement-task`](../recipes/implement-task.md) recipe. Applying it is one reviewed diff, and from then on
every entry and file it wrote is the repository's own.

## What it sets up

| Part | What applying writes |
| --- | --- |
| Ingredients | pins of `git`, `claude`, `codex`, `ralph-tui` and `beads` |
| Stages | `vcs: git`; the workers `claude`, interactive, and `codex`, unattended, with the network |
| Entries | the `hooks` and `verify` modules, a validator, a standing-context entry, `workspaces.update_before_landed`, the unattended denies, three tracker actions |
| Templates | `.ralph-tui/config.toml`, `.ralph-tui/templates/beads.hbs` and `.agents/directives/unattended.md` |
| Recipes | `implement-task`, copied into `.cook/recipes/implement-task/` |
| Detect rule | `.beads/`, `.ralph-tui/` |

A task goes like this. ralph-tui, started in the primary checkout, takes the next ready bead of the epic, renders its
prompt from the station's prompt template and runs the command the configuration names. That command reads the bead's
id from the prompt's first line and runs
`./cook make implement-task --task <id> --unattended --commit --withhold '<promise>COMPLETE</promise>'` with the
prompt on stdin. The run creates the workspace `task-<id>` on `work/task-<id>` from `origin/<default branch>` as the
clone last fetched it, where `start` claims the bead; its `attempt` step starts Codex on the prompt, with the
station's directive before the task's text, under the attempt contract; then the run verifies the
change itself, replays it from the base, commits it and fires `completed`, which closes the bead. Only then does
`the-cook` print the completion tag, and ralph-tui counts the task done. A run that stops fires `blocked`, which marks
the bead blocked with the reason, keeps its workspace for a person and exits non-zero, and ralph-tui skips the task for
the rest of the session.

What the station leaves to the repository: delivery, because pushing and opening pull requests depend on the hosting
platform and the team's convention; a stack's commands and lanes, which a [stack station](node.md) adds; the Beads
database, which `bd init` creates; and installing the programs.

## Ingredients

| Ingredient | Why the station pins it |
| --- | --- |
| [`git`](../ingredients/git.md) | fills the version-control contract, renders the ignore rules of the state paths, and holds git's floor |
| [`claude`](../ingredients/claude.md) | fills the worker contract for Claude Code: its native permissions, hooks and instructions, and its floor |
| [`codex`](../ingredients/codex.md) | fills the worker contract for Codex, the unattended worker: its launch, native rules and floor |
| [`ralph-tui`](../ingredients/ralph-tui.md) | keeps ralph-tui's session files out of version control and checks its version |
| [`beads`](../ingredients/beads.md) | checks the version of `bd`, which the tracker actions run |

Each is pinned from `github:lmarqs/the-cookbook` at `v0.1.0` by the sha256 of its archive. An ingredient the repository
already pins keeps the repository's pin, and the difference is reported
([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)).

## Workers

```yaml
vcs: git
workers:
  claude: { ingredient: claude, role: interactive }
  codex: { ingredient: codex, role: unattended, network: true }
```

- `claude` is `interactive`: people work with it in their sessions, under the shared policy and the floor of every
  ingredient in use.
- `codex` is `unattended` and the repository's only unattended worker, so the `attempt` step of `implement-task`,
  which names no worker, starts it. Its native files also carry `unattended.deny`
  ([roles](https://github.com/lmarqs/the-cook/blob/main/docs/features/workers.md#roles)).
- Neither worker gets `writable_roots`: the worker of an attempt never writes the tracker, so `.beads/` stays outside
  its sandbox.
- `codex` gets `network: true`, because the `implement-task` attempts install dependencies and run tests that need the
  network. With `false`, Codex keeps an attempt's commands off the network
  ([codex](../ingredients/codex.md#a-codex-attempt-cannot-reach-the-network)); a repository whose tasks need no
  network sets `false`.
- No `model` or `effort` is set. Add them under `workers.codex` to choose the model the attempts run with; ralph-tui's
  own model settings never reach the worker.

## Entries

### `.cook/config.yaml`

```yaml
modules: { hooks: true, verify: true }
verify:
  validators:
    - { when: [".cook/**", AGENTS.md], run: ./cook render --check }
context:
  always:
    - { file: .agents/directives/unattended.md, for: [unattended] }
workspaces:
  update_before_landed: true
```

- `modules.hooks` renders the worker hooks: post-edit feedback and `when` entries of standing context for every
  worker. Codex runs them only after a person has reviewed them
  ([codex pitfalls](../ingredients/codex.md#hooks-run-only-after-a-person-reviews-them)).
- `modules.verify` turns on `./cook verify`, its receipt and the
  [`completed` guard](https://github.com/lmarqs/the-cook/blob/main/docs/features/tasks.md#the-completed-guard), so the
  `completed` action closes a bead only after its checks pass.
- The validator proves that a change to `.cook/` or `AGENTS.md` leaves the rendered files in step with it.
- The `context` entry gives every unattended session the directive below, read from the base commit in an attempt, so a
  worker cannot rewrite its own rules. Each task's attempt receives it in its prompt, before the task's text, because
  `./cook make implement-task` passes the prompt from ralph-tui's command to Codex on stdin
  ([delivery in the prompt](https://github.com/lmarqs/the-cook/blob/main/docs/features/context.md#delivery-in-the-prompt)),
  so it reaches Codex though Codex runs no project hook a person has not reviewed.
- `workspaces.update_before_landed` makes `./cook workspace remove` run `git fetch origin` before it checks whether the
  workspace's branch landed. The station's work lands through pull requests merged on the hosting platform, so the
  workspace of a delivered task is removed as landed once the default branch holds it, without a fetch by hand first.

### `.cook/policy.yaml`

```yaml
unattended:
  deny:
    - [git, [am, cherry-pick, commit, merge, push, rebase, reset, stash, tag, update-ref, worktree]]
    - [git, ["-C", "-c"]]
    - [gh]
    - [bd]
    - [ralph-tui]
    - [the-cook, workspace]
```

These deny only attempts; interactive sessions keep the shared policy.

- The `git` subcommands change history or leave the workspace. The run commits a change once its checks pass, and
  anything that leaves the workspace before the replay can be neither checked nor undone.
- `git -C` and `git -c` act on another checkout or override git's configuration.
- `gh` is the hosting platform's command line; delivery is the repository's `deliver` actions, run after the checks.
- `bd` as a whole, because the worker receives the bead in its prompt and the tracker is written by the lifecycle
  actions and by ralph-tui, after the checks. A narrower deny such as `[bd, close]` leaves `bd done` and
  `bd update --status closed`, which close a bead too.
- `ralph-tui`, so a worker never starts a loop runner of its own.
- `the-cook workspace`, because the run creates the workspace, and a worker that creates or removes one acts outside
  its checkout.

### `.cook/hooks.yaml`

```yaml
lifecycle:
  start:
    - name: claim-task
      run: 'BEADS_DIR="$COOK_REPO/.beads" bd update "$COOK_TASK" --status in_progress --assignee "$COOK_ACTOR"'
      auto: true
      scope: [task]
  completed:
    - name: close-task
      run: 'BEADS_DIR="$COOK_REPO/.beads" bd close "$COOK_TASK" --reason "verified on $COOK_BRANCH"'
      auto: true
      scope: [task]
  blocked:
    - name: block-task
      run: 'BEADS_DIR="$COOK_REPO/.beads" bd update "$COOK_TASK" --status blocked --append-notes "blocked: ${COOK_REASON:-no reason given}"'
      auto: true
      scope: [task]
```

- `claim-task` marks the bead in progress and assigns it to the workspace's actor, so the tracker shows which workspace
  holds it.
- `close-task` closes the bead. `completed` with scope `task` is refused without a valid receipt, so the bead closes
  only once the run's replay passed.
- `block-task` marks the bead blocked and appends the reason to its notes. `bd ready` leaves blocked beads out, so no
  later run takes the task while its workspace waits for a person.
- `scope: [task]` keeps the actions out of a workspace without a task, and `auto: true` runs them at the trigger points
  of `the-cook make`
  ([lifecycle](https://github.com/lmarqs/the-cook/blob/main/docs/features/lifecycle.md#trigger-points)).
- `BEADS_DIR="$COOK_REPO/.beads"` makes every action write the primary checkout's database, though the actions run in
  the task's workspace ([beads](../ingredients/beads.md#a-workspace-can-find-a-database-of-its-own-first)).

ralph-tui updates the bead too: it sets it `in_progress` before it starts the command, and closes it with
`bd close <id> --force` once it reads the completion tag. `the-cook make` prints the tag only after `close-task` ran,
so both close the bead after its checks pass.

## Templates

Each is written once, when its file is absent, and is the repository's own file from then on.

### `.ralph-tui/config.toml`

```toml
# Written once by the agentic-coding station of github:lmarqs/the-cookbook, and the repository's own file from then on.
# docs/stations/agentic-coding.md in that cookbook explains each key.
configVersion = "2.1"
tracker = "beads"
agent = "implement-task"
autoCommit = false
maxIterations = 10

[errorHandling]
strategy = "skip"

[parallel]
mode = "never"

[sandbox]
enabled = false

[[agents]]
name = "implement-task"
plugin = "codex"
command = "bash"
defaultFlags = [
  "-c",
  '''
IFS= read -r marker
case "$marker" in
  COOK_TASK_ID=?*) task=${marker#COOK_TASK_ID=} ;;
  *)
    echo "implement-task: the prompt does not start with COOK_TASK_ID=<id>; see .ralph-tui/templates/beads.hbs" >&2
    exit 1
    ;;
esac
exec ./cook make implement-task --task "$task" --unattended --commit --withhold '<promise>COMPLETE</promise>'
''',
  "implement-task",
]

[agents.rateLimitHandling]
enabled = false
```

| Key | Why |
| --- | --- |
| `configVersion = "2.1"` | without it, the first run migrates and rewrites the file ([ralph-tui](../ingredients/ralph-tui.md#a-configuration-without-configversion-is-rewritten-at-the-first-run)) |
| `tracker = "beads"`, `agent = "implement-task"` | the run's tracker and its one agent entry; set in the project's file, so a per-user value does not win |
| `autoCommit = false` | ralph-tui runs in the primary checkout, where a task changes nothing; the run commits in the task's own workspace |
| `maxIterations = 10` | the run's budget: at most ten iterations, one task each, per `ralph-tui run`; `--iterations <n>` sets another |
| `errorHandling.strategy = "skip"` | a run that stopped keeps its workspace, so retrying the task exits 1; skipped, it waits for a person |
| `parallel.mode = "never"` | each task's run makes a workspace of its own from the default branch, so ralph-tui's parallel worktrees would hold nothing to merge |
| `sandbox.enabled = false` | the worker runs in the sandbox its ingredient starts it in; ralph-tui's sandbox would also enclose `the-cook` and the lifecycle actions |
| `plugin = "codex"` | the plugin decides only how ralph-tui detects the command and which arguments it appends; the command reads none of them |
| `command = "bash"` | ralph-tui runs `<command> --version` to detect the agent, which `bash` answers and `dash` refuses |
| `defaultFlags` | the per-task command, below |
| `rateLimitHandling.enabled = false` | a stopped run's text can match ralph-tui's rate-limit patterns, and a retry of its task exits 1 |

ralph-tui has no placeholder for the task id in an agent's command, and passes each task only through the prompt it
renders. The station's prompt template therefore puts the id on the prompt's first line, `COOK_TASK_ID=<id>`, and the
command reads that line and nothing else: a prompt that does not start with it exits 1 before anything runs. The rest
of the prompt reaches `./cook make` on stdin unchanged, and the recipe's `attempt` step, which has no prompt of its
own, passes it to the worker. The arguments ralph-tui's `codex` plugin appends,
`exec --json --sandbox workspace-write -`, become positional parameters the script never reads, so nothing ralph-tui
adds reaches the worker.

| Argument | Why |
| --- | --- |
| `--task "$task"` | binds the run's workspace to the bead: `COOK_TASK` for the actions, `task-<id>` for the workspace name |
| `--unattended` | nobody drives the run, so a handoff stops it as blocked instead of waiting |
| `--commit` | commits the verified change on `work/task-<id>`, which makes the work concluded for an automatic `deliver` |
| `--withhold '<promise>COMPLETE</promise>'` | removes ralph-tui's completion tag from everything the run prints, and prints it once at the end of a completed run |

### `.ralph-tui/templates/beads.hbs`

ralph-tui 0.12 looks for a prompt template in this order: `prompt_template` or `--prompt`, then
`.ralph-tui/templates/<tracker>.hbs` in the directory it runs in, then `~/.config/ralph-tui/templates/<tracker>.hbs`,
then the tracker's built-in one. With the `beads` tracker, the station's template is the second, so it wins over a
per-user one. Its source in the station:

```handlebars
{{=<% %>=}}
COOK_TASK_ID={{taskId}}
# {{taskTitle}}
{{#if taskDescription}}

{{taskDescription}}
{{/if}}
{{#if acceptanceCriteria}}

## Acceptance criteria

{{acceptanceCriteria}}
{{/if}}
{{#if epicId}}

This task belongs to the epic {{epicId}}{{#if prdName}}, {{prdName}}{{/if}}.
{{/if}}
{{#if prdContent}}

## The epic's product requirements

{{prdContent}}
{{/if}}
```

The first line is `the-cook`'s set-delimiter tag
([template language](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-template-language)):
from there on `the-cook` reads `<% %>` as its tags, so it writes ralph-tui's Handlebars `{{ }}` as text, and the file in
the repository starts at `COOK_TASK_ID={{taskId}}`.

| Line | Why |
| --- | --- |
| `COOK_TASK_ID={{taskId}}` | the marker the per-task command reads the bead's id from; ralph-tui trims the prompt, so it is the first line |
| `# {{taskTitle}}` and `{{taskDescription}}` | the bead's title and description, without the `## Acceptance Criteria` section, which ralph-tui gives apart |
| `{{acceptanceCriteria}}` | the criteria ralph-tui reads from that section, or from the description's `- [ ]` lines |
| `{{epicId}}`, `{{prdName}}`, `{{prdContent}}` | the epic the bead belongs to and, when the epic's external reference is `prd:<path>`, the title and text of that file |

The template holds nothing about how the task ends. ralph-tui's built-in Beads prompt tells the worker to close the
bead with `bd close`, to append to `.ralph-tui/progress.md` and to print the completion tag; here the run closes the
bead, the denies refuse `bd`, and `the-cook` prints the tag. ralph-tui offers other values too, such as `labels`,
`priority`, `dependsOn` and `recentProgress`; add the ones your tasks need.

### `.agents/directives/unattended.md`

```markdown
# Rules for unattended work

- Work only on the task in your prompt, in this checkout.
- Run `./cook verify` with the lanes your change needs before you finish: the run runs the lanes of that selection
  itself and replays them from the base.
- Never commit, push or change the tracker: the run commits a change whose checks pass, and its lifecycle actions
  close the task.
- When the task needs a decision only a person can make, stop and say so in one line.
```

The prompt the worker receives holds the task and nothing about how this repository finishes one. The directive says
it: the worker's own `./cook verify` gives the run its selection of lanes, and a lane the lane rules demand that nobody
selected stops the run at Verify.

## Recipes

| Recipe | Why the station adds it |
| --- | --- |
| [`implement-task`](../recipes/implement-task.md) | the kind of change ralph-tui makes for every bead: one task, one workspace, one verified commit |

It is published under the same tag and pinned by the sha256 of its archive. Applying copies it into
`.cook/recipes/implement-task/` with its `origin`; a directory that exists is kept as it is.

## Detect rule

`[.beads/, .ralph-tui/]`: a repository that already uses Beads or ralph-tui fits the station, and
`the-cook scaffold --cookbook` applies it there. `AGENTS.md` alone does not match, because the station pins five
ingredients and adds denies and tracker actions a repository should choose by name.

## Applying it

Read it, and the ingredients it pins, before applying it:

```sh
./cook station show agentic-coding --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook ingredient show codex --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply agentic-coding --cookbook github:lmarqs/the-cookbook@v0.1.0 --dry-run
./cook station apply agentic-coding --cookbook github:lmarqs/the-cookbook@v0.1.0
git add -A && git commit -m "chore: apply the agentic-coding station"
```

A new repository takes it with a stack station at scaffold:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --no-detect --station agentic-coding --station node
```

What to review in the diff:

- the five pins in `ingredients:`, and each `kept ingredients.<name>` line for a pin the repository already holds;
- `vcs:` and the two workers, and a `kept` line for a worker of the same name the repository already declares;
- the unattended denies, against what the repository's attempts need;
- the three tracker actions: they run `bd` with the credentials of whoever fires the event;
- `.ralph-tui/config.toml`: every task's command still runs `./cook make implement-task` with `--unattended`, and no
  key turns a permission check off; a `kept .ralph-tui/config.toml: exists` line means the repository's own file stays
  as it is, and its agent entries still need a command that runs `the-cook`, as the template's does;
- `.ralph-tui/templates/beads.hbs`: its first line is `COOK_TASK_ID={{taskId}}`, and nothing in it asks the worker to
  touch the tracker; `ralph-tui template show` prints the template a run uses and the file it comes from;
- `.cook/recipes/implement-task/recipe.yaml`, with its `origin`;
- the renders of the new pins: the workers' native files, the floor rules in the `policy` block of `AGENTS.md`, and the
  `.gitignore` blocks.

Then, on each machine, `./cook fetch`, and `bd init --skip-hooks --skip-agents` where the clone has no Beads database.
To take a newer tag's entries later, apply it again with `--dry-run` first
([applying again](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#applying-again)).

## Pitfalls

### Every task starts from the default branch

- **Symptom.** A bead that builds on another bead's change fails, or redoes that change.
- **Why.** Each run creates its workspace from the default branch, and a bead whose dependency closed is ready before
  the dependency's branch is merged.
- **What the station does.** Nothing. Merge each change before the beads that need it run, or run them on one branch
  as in [Run stories that build on one another on one branch](../use-cases/dependent-stories-one-branch.md).

### A prompt without the marker stops every task

- **Symptom.** Every task fails with `implement-task: the prompt does not start with COOK_TASK_ID=<id>`, and nothing is
  created.
- **Why.** The command reads the id only from the first line of the prompt. A `prompt_template` or `--prompt` takes
  precedence over `.ralph-tui/templates/beads.hbs`, and `--prd`, `--tracker beads-bv` or another tracker reads another
  file, `<tracker>.hbs`, or a built-in prompt.
- **What the station does.** It stops before anything runs. Keep `COOK_TASK_ID={{taskId}}` as the first line of every
  prompt template the runs use, and start ralph-tui from the directory that holds `.ralph-tui/templates/`.

### A run that stops before it starts marks the bead blocked

- **Symptom.** A bead shows as blocked and no workspace `task-<id>` exists.
- **Why.** An unattended run that exits 3 before it creates anything, for a requirement it lacks or an ingredient
  that is not cached, fires `blocked` too, and `block-task` marks the bead.
- **What the station does.** Nothing. Fix what the bead's notes name, such as with `./cook fetch`, then reopen it:
  `bd update <id> --status open`.

### An interrupted run leaves a task its workspace still holds

- **Symptom.** After an interrupted `ralph-tui run`, the next run offers the same bead and skips it at once.
- **Why.** ralph-tui sets the tasks it holds back to open when interrupted, while the recipe run keeps its workspace
  and branch, so a second run of the task exits 1.
- **What the station does.** Nothing. Continue the run with `./cook make --continue task-<id>`, or remove the workspace
  and delete its branch `work/task-<id>`.

### Removing `the-cook` leaves a command that names `./cook`

- **Symptom.** After `the-cook remove`, ralph-tui cannot start a task.
- **Why.** Both removal modes delete the wrapper, and keep the templates a station wrote, so `.ralph-tui/config.toml`
  still names `./cook make implement-task`.
- **What the station does.** Nothing: the file is the repository's own. Point the agent entry at the program the team
  runs without `the-cook`, or delete the entry.

### ralph-tui must start in the repository's root

- **Symptom.** Every task fails because `./cook` is not found.
- **Why.** ralph-tui starts the command in its own working directory, and the command names `./cook` relative to it;
  ralph-tui also looks for `.ralph-tui/templates/beads.hbs` there.
- **What the station does.** Nothing. Start ralph-tui from the root of the primary checkout, or pass `--cwd` with it.
