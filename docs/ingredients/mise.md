# mise

The `mise` ingredient describes mise, a tool-version manager and task runner, to `the-cook`. Pinned, it keeps workers
from chaining an unlisted task onto an allowed one, and `doctor` checks the mise version on each machine that runs it.

## What it describes

- **Program.** [mise](https://mise.jdx.dev), the 2026.6 releases. Checked against mise 2026.6.13.
- **Contract.** None. The lifecycle reaches mise through the repository's own commands, lanes, validators, `bootstrap`,
  lifecycle actions and recipe `run` steps, such as `mise run lint {file}`, which `the-cook` runs as written.
- **Detect.** `mise.toml`, `.mise.toml`, and the directories mise reads task scripts from: `mise-tasks/`,
  `.mise-tasks/`, `.mise/tasks/`, `mise/tasks/` and `.config/mise/tasks/`.

`the-cook` contains no code for mise. It never reads or writes `mise.toml`, `.mise.toml`, `.tool-versions` or a task
file, and nothing in it needs mise: the wrapper `./cook` runs without it. A `verify` task the repository keeps in mise
stays as it is, and `the-cook verify` adds nothing to it. `doctor` reports `mise` missing on a machine when a declared
command or action names it. No station of this cookbook lists the ingredient; a repository pins it on its own with
`the-cook ingredient pin`.

## Floor

One deny rule, a bare string:

```yaml
- "mise * ::: *"
```

`:::` separates tasks on one mise command line: `mise run lint ::: test ::: check` runs the three tasks in parallel. A
prefix rule matches the leading words only, so an `allow` of `[mise, run, test]` would also let
`mise run test ::: deploy` start `deploy`. The `*` before `:::` covers every way mise starts a task: `mise run`, its
alias `mise r`, `mise tasks run`, and `mise <task>`, mise's shorthand for `mise tasks run <task>`.

A bare string is a whole-command glob, so the rule renders only for a worker whose program matches whole shell commands
by glob: a worker of the [`claude`](claude.md) ingredient, not one of the [`codex`](codex.md) ingredient. The `policy`
block of `AGENTS.md` lists it, marked as a floor rule of the mise ingredient and as not enforced in each worker that
leaves it out. As [floor](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-floor), it is
never written into `.cook/policy.yaml`, no policy entry removes it, and it holds only while the ingredient is pinned:
a repository that runs mise without the ingredient has no `:::` deny.

## Checks

| Check | Value |
| --- | --- |
| `version` | `mise --version`, which prints `2026.6.13 linux-x64 (2026-06-23)`; the range is `>=2026.6 <2026.7` |
| `detect` | the configuration files and task directories above |

mise prints its version on stdout and its notices about a newer release on stderr. A mise outside 2026.6 is a
`doctor` finding until a tag of this cookbook widens the range.

## Pitfalls

### Task chaining lets an allowed task carry an unlisted one

- **Symptom.** `[mise, run, test]` is allowed, and `mise run test ::: deploy` runs `deploy` without a prompt.
- **Why.** A prefix rule matches the leading words, `mise run test`, and `:::` starts another task on the same command
  line.
- **What the ingredient does.** Its floor denies every mise command line with `:::` in each worker whose program can
  match it. A worker that needs several tasks starts each on its own command line, where each meets its own rule.

### A task runs whatever its definition says

- **Symptom.** An allowed `mise run deliver` runs `the-cook lifecycle run deliver`, which the policy puts under `ask`,
  without a prompt.
- **Why.** A rule matches the command the worker starts, not the commands a task runs: a task's `run`, its script and
  its dependencies are code in the repository.
- **What the ingredient does.** Nothing beyond its floor. Allow a task by name only after reading it and the tasks it
  depends on, and review changes to `mise.toml` and the task directories like code. The binary's own guards, such as
  the `deliver` guard and its refusal of `the-cook lifecycle run` inside an attempt, hold whatever task starts the
  command.

### The floor does not hold in a codex worker

- **Symptom.** In a session of a worker of the `codex` ingredient, an allowed `mise run test ::: deploy` runs
  `deploy`.
- **Why.** The rule is a whole-command glob with `:::` in the middle, and codex's rules match command prefixes only.
- **What the ingredient does.** Renders the rule for every worker that can enforce it, and the `policy` block marks
  the gap. In a codex worker, a deny of a task by name, such as `[mise, run, deploy]`, still stops `mise run deploy`
  started on its own line, but not the same task chained after an allowed one.

### A workspace loads the primary checkout's mise configuration as well

- **Symptom.** In a workspace, a mise task, tool or environment value comes from the primary checkout: a task the
  workspace's files do not define, or a value from the primary checkout's local files.
- **Why.** mise reads every configuration file from the current directory up through its parents and merges the
  tasks of all of them, and a managed workspace lives inside the primary checkout.
- **What the ingredient does.** Nothing. Keep the committed configuration complete, so a workspace's own copy defines
  everything it runs, and keep personal values out of files a workspace would inherit. `mise config ls` in the
  workspace lists every file in use.

### A workspace's `mise.toml` is not trusted

- **Symptom.** `bootstrap`, a lane or an attempt in a new workspace fails with an untrusted-config error, mise asks
  whether to trust the file, or the workspace runs without the tools and tasks its file declares.
- **Why.** mise checks trust before it parses a configuration file that can run code or change the environment, and
  records trust per file. The workspace's copy is a file at a new path: mise may skip it, and without a terminal to ask,
  it fails. Configuration that holds only plain tool versions and tasks needs no trust, and in a detected CI mise
  assumes trust unless its paranoid mode is on.
- **What the ingredient does.** Nothing. On each machine, list the clone's workspaces directory in mise's
  `trusted_config_paths` setting, under which mise trusts every configuration file without asking, or run
  `mise trust mise.toml` in each new workspace.

### A second pin of `the-cook` in mise drifts from the wrapper

- **Symptom.** The repository's `mise.toml` or `.tool-versions` names one release of `the-cook`, `.cook/wrapper.yaml`
  another, and an upgrade changes only one of them.
- **Why.** Two files answer the same question. Inside the repository a `the-cook` that mise puts on `PATH`
  [defers](https://github.com/lmarqs/the-cook/blob/main/docs/features/wrapper.md#a-global-install-defers-to-the-pin) to
  the pin, so the mise entry decides nothing and misleads whoever reads it.
- **What the ingredient does.** Nothing: `the-cook` reads its version only from `.cook/wrapper.yaml` and never reads
  mise's files. Remove `the-cook` from the repository's mise tools; a global install,
  `mise use -g github:lmarqs/the-cook@<tag>`, lives in your global mise configuration instead.

## Examples

### Pin it

```sh
./cook ingredient list --cookbook github:lmarqs/the-cookbook@v0.1.0   # mise: detected, not pinned
./cook ingredient show mise --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook ingredient pin mise --cookbook github:lmarqs/the-cookbook@v0.1.0
git diff
```

```yaml
# .cook/config.yaml, the ingredients section
ingredients:
  - name: mise
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

The render adds the `:::` deny to the permissions of each worker that can enforce it and lists it in the `policy`
block of `AGENTS.md`.

### Commands and lanes that run mise tasks

The stack stations of this cookbook call each stack's own tools, so they work without mise. A repository that keeps
its commands in mise tasks edits the applied entries to run them:

```yaml
# .cook/config.yaml, the commands and verify sections
commands:
  lint: mise run lint {file}
  build: mise run build
verify:
  lanes:
    unit: { select: "mise run test {pattern}", full: mise run test }
```

### Policy entries that name tasks

```yaml
# .cook/policy.yaml
schema: 1
shared:
  allow:
    # Tasks whose effects stay in this clone.
    - [mise, run, [lint, test, build]]
    # This clone's local services, which a person working with a worker resets and restarts.
    - [mise, run, ["db:reset", "docker:up", "docker:down"]]
  ask:
    - [the-cook, lifecycle, run, deliver]
  deny: []
unattended:
  deny:
    # Shared local services another attempt may be using.
    - [mise, run, ["db:reset", "docker:down"]]
```

### One lint command for two stacks

`commands.lint` is one command, so a repository with two stacks runs a task that picks the linter by the file's
extension. An executable script in `.mise/tasks/lint` is the task `lint`; its `#USAGE` line declares the argument
mise passes as `usage_file`:

```sh
#!/usr/bin/env sh
#MISE description="Lint one file with the linter of its stack"
#USAGE arg "[file]" help="The file to lint"
set -eu
case "${usage_file:-}" in
  *.ts | *.js) npx eslint "$usage_file" ;;
  *.py) ruff check "$usage_file" ;;
  *) echo "no linter for ${usage_file:-}" ;;
esac
```

```yaml
# .cook/config.yaml, the commands section
commands:
  lint: mise run lint {file}
```

### Run `the-cook` through mise

mise is one way to get a `the-cook` binary on a machine, to scaffold a repository that has no `cook` yet or as a global
install
([installation](https://github.com/lmarqs/the-cook/blob/main/docs/features/installation.md#global-install-optional)):

```sh
mise exec github:lmarqs/the-cook@<tag> -- the-cook scaffold    # once, nothing installed globally
mise use -g github:lmarqs/the-cook@<tag>                        # global install, in ~/.config/mise/config.toml
```

Inside an adopted repository, `./cook` runs the release `.cook/wrapper.yaml` pins, with or without mise.
