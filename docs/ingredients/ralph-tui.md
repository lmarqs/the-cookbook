# ralph-tui

The `ralph-tui` ingredient describes [ralph-tui](https://github.com/subsy/ralph-tui), a loop runner, so that a
repository that runs it keeps ralph-tui's session files out of version control and `./cook doctor` checks its version.
It fills no contract: `the-cook` never starts ralph-tui; ralph-tui starts a `the-cook` command for each task.

## What it describes

| | |
| --- | --- |
| Program | ralph-tui, a loop runner that takes the tasks of a tracker one at a time and starts an agent command for each |
| Versions | `>=0.12 <0.13`; the facts on this page were checked against ralph-tui 0.12.0 |
| Contract | none |
| Detect | `.ralph-tui/` |

ralph-tui reads tasks through tracker plugins: `beads` and its variants `beads-bv`, `beads-rust` and `beads-rust-bv`,
`json` for a JSON list of user stories that ralph-tui calls a PRD, `jira` and `linear`. For each task it renders a
prompt from a Handlebars template, starts the command of an agent entry with the prompt on stdin, and counts the task
done when the command's standard output holds `<promise>COMPLETE</promise>`. When it counts a task done, its tracker
plugin closes the task itself: the Beads plugin runs `bd close <id> --force`.

Its configuration is TOML: `.ralph-tui/config.toml`, found from the working directory upward, merged over the per-user
`~/.config/ralph-tui/config.toml`. The project's values win, an array such as `[[agents]]` replaces the user's, and a
table such as `[sandbox]` merges key by key. Unknown keys are a configuration error. `ralph-tui config show --sources`
prints the merged result and the files it came from.

An agent entry names a `plugin`, the program ralph-tui was written to drive, and may override its `command`. ralph-tui
starts `command` with the entry's `defaultFlags` first, then the arguments its plugin adds for that program, and writes
the prompt on stdin. That is how a loop runner reaches `the-cook`: an entry whose command runs
`./cook make <recipe> --task <id> --unattended` or `./cook attempt <worker>`
([attempts](https://github.com/lmarqs/the-cook/blob/main/docs/features/attempts.md#how-a-loop-runner-calls-it)).
ralph-tui has no placeholder for the task id in `command` or `defaultFlags`: the only per-task values are those its
prompt template reads, such as `{{taskId}}`, `{{taskTitle}}`, `{{taskDescription}}` and `{{acceptanceCriteria}}`. It
takes the first template it finds of `prompt_template` or `--prompt`, `.ralph-tui/templates/<tracker>.hbs` in the
directory it runs in, `~/.config/ralph-tui/templates/<tracker>.hbs`, and the tracker's built-in one, and
`ralph-tui template show` prints the one in use with its source. The
[agentic-coding station](../stations/agentic-coding.md#ralph-tuitemplatesbeadshbs) therefore writes a prompt template
whose first line carries the id, for its command to read.

`the-cook` reads neither configuration file and never writes one after the station's template seeded the project's.
An agent entry whose command does not run `the-cook`, or a script that starts an agent directly, runs outside the
attempt contract, and `the-cook` neither detects nor prevents it.

The ingredient holds the facts that hold in every repository: the ignore rules for ralph-tui's files and the supported
versions. The configuration is a practice, so the [agentic-coding station](../stations/agentic-coding.md) writes it
once, as the repository's own file.

## Renders

The `ralph-tui` block of `.gitignore`:

```gitignore
/.ralph-tui/*
!/.ralph-tui/config.toml
!/.ralph-tui/templates/
```

ralph-tui writes its session (`session.json`, `ralph.lock`), its progress file (`progress.md`), its iteration logs
(`iterations/`) and its reports under `.ralph-tui/` of the directory it runs in. Ignored, they never make a checkout
look changed: `the-cook attempt` refuses a checkout with untracked files that are not ignored, and `--commit` would
commit them. The block keeps the configuration and the project's prompt templates tracked, so a change to either is a
reviewed diff; ralph-tui reads `.ralph-tui/templates/<tracker>.hbs` before its built-in template. Every `./cook render`
writes the block, and `./cook render --check` reports an edit to it.

## Checks

| Check | Value |
| --- | --- |
| `version` | `ralph-tui --version`, which prints `ralph-tui 0.12.0`; supported: `>=0.12 <0.13` |
| `detect` | `.ralph-tui/` |

## Pitfalls

### A task counts as done whatever the exit status

- **Symptom.** A task whose command exited non-zero is closed in the tracker.
- **Why.** ralph-tui decides completion only from `<promise>COMPLETE</promise>` in the command's standard output,
  matched in any letter case and with any spacing inside the tags. The exit status marks the iteration failed and
  nothing more, and the tracker plugin closes the task.
- **What the ingredient does.** Nothing: completion is ralph-tui's. Every per-task command passes
  `--withhold '<promise>COMPLETE</promise>'` to `the-cook`, which removes that text from the output of work that did
  not pass its checks
  ([attempts](https://github.com/lmarqs/the-cook/blob/main/docs/features/attempts.md#the-completion-signal)).
  `--withhold` removes the exact text only, so the prompt template of the
  [agentic-coding station](../stations/agentic-coding.md#ralph-tuitemplatesbeadshbs) never asks the worker to print
  the tag, and `the-cook make` prints it itself once a run completes.

### Agent plugins turn permission checks off by default

- **Symptom.** With a plugin's defaults, ralph-tui starts Claude Code with `--dangerously-skip-permissions` and Codex
  with `--full-auto`.
- **Why.** The `claude` plugin's `skipPermissions` option and the `codex` plugin's `fullAuto` option default to `true`,
  and the plugin adds those arguments after `defaultFlags`.
- **What the ingredient does.** Nothing. The station's per-task command drops every argument the plugin adds. A command
  that forwards them to `the-cook` has each argument the worker's ingredient marks as a floor bypass refused at the
  first task ([the floor](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-floor)).

### An agent named on the command line can start its program directly

- **Symptom.** A run started with `--agent claude` runs Claude Code itself, outside `the-cook`.
- **Why.** `--agent <name>` takes the agent entry whose name or plugin matches. When no entry does and a plugin has that
  name, ralph-tui builds an entry from the plugin's defaults. A per-user `agent = "<name>"` does the same when the
  project's file sets no `agent`.
- **What the ingredient does.** Nothing. The station's configuration sets `agent` to its own entry, so a per-user value
  does not win. Pass `--agent` only with the name of an entry of `.ralph-tui/config.toml`, or not at all.

### A configuration without `configVersion` is rewritten at the first run

- **Symptom.** After the first `ralph-tui run`, `.ralph-tui/config.toml` shows as changed, and ralph-tui's skills appear
  in the user's agent directories.
- **Why.** ralph-tui 0.12 migrates a project configuration whose `configVersion` is absent or older than `2.1`: it
  installs its bundled skills globally for every agent program it detects, copies its prompt templates into
  `~/.config/ralph-tui/templates/` where they are missing, and saves the configuration with `configVersion = "2.1"`.
- **What the ingredient does.** Nothing. The station's template sets `configVersion = "2.1"`; a configuration written by
  hand needs the same line.

### The per-user configuration reaches every run

- **Symptom.** A run behaves differently on one machine.
- **Why.** `~/.config/ralph-tui/config.toml` is merged under the project's file, and every key the project leaves unset
  comes from it, such as `sandbox`, `parallel` or `prompt_template`. A prompt template in
  `~/.config/ralph-tui/templates/` takes precedence over the tracker's built-in one, though not over the project's
  `.ralph-tui/templates/`.
- **What the ingredient does.** Nothing: `the-cook` reads neither file. The station's template sets the keys its runs
  depend on, and `ralph-tui config show --sources` shows where the others come from.

### API keys do not reach the agent's command

- **Symptom.** A worker that logs in with an API key fails to authenticate under ralph-tui, and works when started by
  hand.
- **Why.** ralph-tui removes every variable named like `*_API_KEY`, `*_SECRET_KEY` or `*_SECRET` from the environment of
  the command it starts, unless the agent entry lists it in `envPassthrough`. Everything that command starts inherits
  the filtered environment: `the-cook`, the worker and the lifecycle actions.
- **What the ingredient does.** Nothing. Add the variable the worker needs to the agent entry, such as
  `envPassthrough = ["OPENAI_API_KEY"]`.

### ralph-tui runs the command with `--version` before the first task

- **Symptom.** `Agent 'codex' not available` before any task starts.
- **Why.** Before a run, ralph-tui starts the agent entry's `command` with `--version` to detect it, and the `codex`
  plugin also needs a dotted version number in the output. A shell such as `dash`, which is `/bin/sh` on Debian and
  Ubuntu, rejects `--version`.
- **What the ingredient does.** Nothing. The station's entry runs `bash`, which prints its version.

### Live output stays empty

- **Symptom.** The TUI and the headless log show nothing while a task runs.
- **Why.** ralph-tui's `claude` and `codex` plugins display only the JSON events of the programs they were written for,
  and the text `the-cook` prints is not one of them.
- **What the ingredient does.** Nothing. The whole output of each task is in `.ralph-tui/iterations/`, and
  `./cook make --status <workspace>` prints the record of a recipe run.

### A failed command can read as a rate limit

- **Symptom.** ralph-tui waits, then runs a failed task's command again.
- **Why.** ralph-tui matches the command's standard error against rate-limit patterns such as `rate limit`,
  `too many requests`, `overloaded` and `429`, whatever wrote the text, such as a failing test, and retries with
  backoff when one matches.
- **What the ingredient does.** Nothing. The station's entry turns `rateLimitHandling` off: a recipe run that stopped
  keeps its workspace, so a second run of the same task exits 1.

### A run without `--epic` takes every ready task

- **Symptom.** A Beads run picks up tasks from other work.
- **Why.** Without `--epic`, the Beads plugin asks `bd ready` for ready tasks across the whole database.
- **What the ingredient does.** Nothing. Pass `--epic <id>`.

### Acceptance criteria come from the description

- **Symptom.** A task's prompt has no acceptance criteria, although the bead has some.
- **Why.** ralph-tui reads acceptance criteria from a `## Acceptance Criteria` section, or from `- [ ]` lines, of the
  task's description. Its Beads plugin does not read the bead's acceptance field.
- **What the ingredient does.** Nothing. End each description with a `## Acceptance Criteria` section, as a `- [ ]`
  checklist.

### ralph-tui's model and effort settings do not fit Claude Code

- **Symptom.** A full Claude model id is refused, and `--variant` changes nothing.
- **Why.** ralph-tui's `claude` plugin accepts only the aliases `sonnet`, `opus` and `haiku` as a model, and neither its
  `claude` nor its `codex` plugin passes an effort.
- **What the ingredient does.** Nothing. Set the worker's `model` and `effort` in the profile's `workers.<name>`, which
  `the-cook` passes at every launch as the worker's ingredient maps them; the station's per-task command passes none of
  ralph-tui's arguments to the worker.

### Parallel workers run beside the repository

- **Symptom.** `ralph-tui run --parallel` creates checkouts outside the repository, and branches it never had.
- **Why.** In parallel mode ralph-tui runs each task in a git worktree of its own, by default in
  `../.ralph-worktrees/<project>/` beside the checkout, on a branch `ralph-parallel/<session>/<task>`, and merges the
  results onto a session branch `ralph-session/<id>`. It force-removes a stale worktree at a worker's path before
  reusing the path, and its scheduler ignores dependencies on tasks outside the run.
- **What the ingredient does.** Nothing. `the-cook` never cleans ralph-tui's worktrees or branches, and their names do
  not collide with the `work/<name>` branches of managed workspaces.

### Interrupting a run reopens its tasks

- **Symptom.** After an interrupt, the task that was running is ready again.
- **Why.** When interrupted, ralph-tui sets the tasks it holds in progress back to open.
- **What the ingredient does.** Nothing. For a task that a recipe run was making, see the station's
  [pitfalls](../stations/agentic-coding.md#pitfalls).

## Examples

### Pinning it

```sh
./cook ingredient pin ralph-tui --cookbook github:lmarqs/the-cookbook@v0.1.0
```

```yaml
# .cook/config.yaml
ingredients:
  - name: ralph-tui
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

Pinning renders the `.gitignore` block and lets `./cook doctor` check the version. The ingredient declares no worker
and writes no `vcs:`; the [agentic-coding station](../stations/agentic-coding.md) pins it with the others it uses and
writes the configuration.

### Reading the configuration a run uses

```sh
ralph-tui config show --sources
```

Run it before the first run on a machine: it shows whether `~/.config/ralph-tui/config.toml` sets a key the project's
file leaves unset.

### Stopping using it

```sh
# delete the "name: ralph-tui" entry from ingredients: in .cook/config.yaml
./cook render        # removes the ralph-tui block from .gitignore
git diff             # .ralph-tui/config.toml and the unattended denies stay, as the repository's own
```

### A configuration of your own

A repository that runs ralph-tui without the station writes the agent entry itself, with `configVersion = "2.1"`, an
`agent` that names the entry, and a per-task command that passes `--withhold '<promise>COMPLETE</promise>'` to
`the-cook`. [Run stories that build on one another on one branch](../use-cases/dependent-stories-one-branch.md) shows
an entry that runs each task as one attempt.
