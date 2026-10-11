# python

The `python` station gives a Python project the syntax check, the `unit` lane run with pytest and the check-on-edit
hook that `the-cook` runs at the Verify stage and after each edit a worker makes. Its entries call `python3`, its
standard library and pytest as a module of it, so they work in a repository that uses no task runner and no project
manager.

## What it sets up

`python` is a stack station: it pins no ingredient, declares no stage, worker or `vcs`, and holds no template and no
recipe. Applied, it adds entries to `.cook/config.yaml` and `.cook/hooks.yaml` and nothing else, so it adds a command
and a lane to a lifecycle that another station, such as [`agentic-coding`](agentic-coding.md), or the repository itself
sets up. The lane acts once `modules.verify` is on, and the post-edit hook once `modules.hooks` is on
([modules](https://github.com/lmarqs/the-cook/blob/main/docs/features/profile.md#modules)).

```yaml
schema: 1
name: python
version: v0.1.0
description: the syntax check, pytest lane and post-edit hook of a Python project
detect: [pyproject.toml, requirements.txt]
entries:
  .cook/config.yaml:
    commands:
      lint: python3 -m ast {file} > /dev/null
    verify:
      app_paths: ["**/*.py", pyproject.toml, setup.cfg, pytest.ini, "requirements*.txt"]
      lint_paths: ["**/*.py"]
      lanes:
        unit:
          select: python3 -B -m pytest {pattern}
          full: python3 -B -m pytest
  .cook/hooks.yaml:
    post_edit:
      - paths: ["**/*.py"]
        run: python3 -m ast {file} > /dev/null
```

The entries are checked against Python 3.13.14 and pytest 8.3.5. A station holds no version check, because only an
ingredient has checks: `./cook doctor` reports `python3` missing on a machine that lacks it, because a declared command
names it.

The station declares no `bootstrap` and no `commands.build`. Python projects install their dependencies in several
ways, a virtual environment with pip, uv or another manager, so the commands run under the `python3` first on `PATH`:
the environment active where the work runs. A repository that manages its environment with uv gives every workspace
its own ([Applying it](#applying-it)). A Python project has no compile step every project shares, and the syntax check
covers each changed file.

## Entries

### `.cook/config.yaml`

| Entry | Value | Why |
| --- | --- | --- |
| `commands.lint` | `python3 -m ast {file} > /dev/null` | It parses the file with the interpreter's own parser and writes nothing: it exits 1 with the `SyntaxError` and its position when the file does not parse, and 0 otherwise. Python ships no linter, so a syntax check is what every Python project shares; `py_compile` would write a bytecode file under `__pycache__`, whatever `-B` says |
| `verify.app_paths` | Python sources and the files that declare the project, its dependencies and pytest's configuration | A change to any of them can change what a test sees, so it is an application change and every lane is selected or skipped ([lane rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/verify.md#lane-rules)) |
| `verify.lint_paths` | `**/*.py` | Every changed Python file goes through `commands.lint` |
| `verify.lanes.unit` | `python3 -B -m pytest {pattern}`, and `python3 -B -m pytest` in full | `{pattern}` is a test file, a directory, or a test's node id, such as `tests/test_invoice.py::test_rounding`. pytest exits 5 when it collects no test, so a pattern that selects nothing fails the lane. `-B` stops Python writing `.pyc` files on import, the modules pytest rewrites for its assertions included, so a lane leaves no `__pycache__` directory behind; pytest's own `.pytest_cache` holds a `.gitignore` that ignores everything in it |

`python3 -B -m pytest` run in the primary checkout never collects the managed workspaces under `.work/workspaces/`:
pytest's default `norecursedirs` skips every directory whose name begins with `.`.

### `.cook/hooks.yaml`

The post-edit hook runs the same syntax check on each `.py` file a worker edits; on a file that does not parse, the
`SyntaxError` and its position go to the worker
([post edit](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#post-edit)). A repository with a
linter or a formatter of its own replaces the command with it, keeping it to the one file and letting it fix what it
can.

### Policy, lifecycle actions and standing context

The station adds none of them. What a worker runs without a prompt, what an attempt must never run, how work is
delivered and which guidance every session receives are choices each repository makes in its own files. Two entries
need more thought than they look:

- `[python3]` under `allow` lets a worker run any code at all;
- `[python3, -m, pytest]` under `allow` lets a worker run the tests and the `conftest.py` files it just edited, which
  pytest loads on every run ([the-cook pitfalls][task-runner]).

## Detect rule

`detect: [pyproject.toml, requirements.txt]`: either file at the root of the repository. The scaffold applies the
station when one matches, and `the-cook station list --cookbook` shows whether it does
([station detection](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#station-detection)). A
`requirements.txt` that only pins a documentation tool, or a `pyproject.toml` that only configures a tool of another
stack, matches too: pass `--no-detect` with a `--station` for each station the repository wants, or delete the station's
entries from `.cook/config.yaml` and `.cook/hooks.yaml`, and its record from `stations:`, after the scaffold.

## Applying it

On a repository without `the-cook`, the scaffold applies `python` when `pyproject.toml` or `requirements.txt` is
there, beside the stations named:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --dry-run
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding
```

On a repository that already uses `the-cook`, read the station, then apply it:

```sh
./cook station show python --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply python --cookbook github:lmarqs/the-cookbook@v0.1.0 --dry-run
./cook station apply python --cookbook github:lmarqs/the-cookbook@v0.1.0
git add -A && git commit -m "chore: apply the python station"
```

`station apply` adds only what the files lack and shows it as one diff
([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)). In the diff,
review:

- every `kept` line: a `commands.lint` or a `unit` lane the repository already names stays as it is;
- `commands.lint` and the hook: replace the syntax check with the repository's own linter, keeping it to the one file;
- the `unit` lane: it runs pytest under the `python3` first on `PATH`, so the environment that holds the project's
  dependencies must be active wherever work runs, or the lane must name it;
- the record the profile gains:

```yaml
stations:
  - name: python
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

A repository managed with uv gives every workspace an environment of its own, and runs the lane in it:

```yaml
bootstrap: [uv sync --locked]
verify:
  lanes:
    unit:
      select: uv run --locked python -B -m pytest {pattern}
      full: uv run --locked python -B -m pytest
```

`uv sync` looks for the project from the current directory upward, so in a workspace it finds the workspace's own
`pyproject.toml`, and creates the project's `.venv` there when it is absent. `--locked` makes it exit with an error
when `uv.lock` is missing or out of date, instead of writing a new one into a fresh workspace. The `.venv/` directory
must be ignored, like any directory a bootstrap creates.

A change then verifies with the tests that exercise it:

```sh
./cook verify --lane unit=tests/test_invoice.py
```

Across many repositories, `the-cook fleet` applies it in each and delivers each change through that repository's own
`deliver` actions ([fleet](https://github.com/lmarqs/the-cook/blob/main/docs/features/fleet.md)):

```sh
the-cook fleet station apply python --cookbook github:lmarqs/the-cookbook@v0.1.0 --yes --deliver
```

The station has no renders and no floor, so nothing it wrote changes with a new release of `the-cook` or a new tag of
this cookbook; a newer tag's entries reach the files only when the station is applied again
([applying again](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#applying-again)).

## Pitfalls

### `python3 -m pytest` finds no pytest

- **Symptom.** The lane fails with `No module named pytest`, while `pytest` runs in the same shell.
- **Why.** `python3 -m pytest` runs pytest as a module of the first `python3` on `PATH`, such as an active virtual
  environment's, and that interpreter lacks it; the `pytest` command on `PATH` can belong to another interpreter.
- **What the station does.** Runs pytest under `python3`, so the lane uses the interpreter, and the dependencies, of
  the environment where the work runs.
- **What you do.** Install pytest in the project's environment, or name that environment's interpreter in the lane.

### A workspace imports the primary checkout's code

- **Symptom.** Tests in a workspace pass or fail against code the workspace does not hold.
- **Why.** A workspace has no environment of its own unless its bootstrap creates one, so its commands use the
  environment active in the shell, usually the primary checkout's ([the-cook pitfalls][inherit]). `python3 -m` puts the
  current directory first on `sys.path`, so a package at the workspace's root wins; a package under `src/`, installed in
  editable mode from the primary checkout, resolves to the primary checkout's copy.
- **What the station does.** Runs every command from the root of the checkout it verifies, with `python3 -m`.
- **What you do.** In a `src/` layout, set pytest's `pythonpath` option to `src`, or give every workspace its own
  environment in `bootstrap`, as the uv example above does.

### A command without `-B` changes the tree

- **Symptom.** `verify` fails because files changed while it ran, or leaves untracked `__pycache__` directories, in a
  repository that replaced the lane with its own command.
- **Why.** Python writes a `.pyc` file under `__pycache__` beside each module it imports, and pytest writes the test
  modules it rewrites the same way, unless `-B` or `PYTHONDONTWRITEBYTECODE` is set. A file no ignore rule covers is a
  new untracked file, and `verify` refuses to certify content that changed under it.
- **What the station does.** Passes `-B` to every interpreter it starts, and checks syntax with `ast`, which writes
  nothing.
- **What you do.** Keep `-B` in the repository's own commands, or ignore `__pycache__/` in `.gitignore`.

### A `norecursedirs` setting collects the workspaces

- **Symptom.** The full lane, run in the primary checkout, collects every test twice or more and fails on duplicate
  module names.
- **Why.** A `norecursedirs` value in the repository's pytest configuration replaces pytest's default list, and with
  it the `.*` pattern that keeps pytest out of `.work/`.
- **What the station does.** Nothing: it never edits the project's tool configuration.
- **What you do.** Keep `.*` in the repository's `norecursedirs`, or list `.work`, or the `state.dir` the profile
  names, in it.

### A full lane fails in a project without tests

- **Symptom.** `./cook verify --full` fails on the `unit` lane with exit status 5.
- **Why.** pytest exits 5 when it collects no test, which is what makes a pattern that selects nothing fail.
- **What the station does.** Leaves pytest's exit status as it is.
- **What you do.** Add the project's first test, or delete the lane until there is one.

### One lint command in a polyglot repository

- **Symptom.** In a repository that applied another stack station first, `verify` lints `.py` files with that stack's
  linter, or the Python tests never run.
- **Why.** `commands.lint` and a lane are named items: the first station applied writes them, and the next one's are
  reported as kept, while the path lists of both are added.
- **What the station does.** Names its lane `unit`, as every stack station of this cookbook does.
- **What you do.** Write one lint command that picks the linter by the file's extension, and declare the Python lane
  under a name of its own, as the [go station shows](go.md#one-lint-command-and-one-unit-lane-in-a-polyglot-repository).

[task-runner]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-task-runner-runs-whatever-its-tasks-say
[inherit]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#workspaces-inherit-the-parents-tool-configuration-and-dependencies
