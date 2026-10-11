# go

The `go` station gives a Go module the lint and build commands, the `unit` lane, the module check and the
format-on-edit hook that `the-cook` runs at the Verify stage and after each edit a worker makes. Its entries call the
Go toolchain's own `go` and `gofmt`, so they work in a repository that uses no task runner.

## What it sets up

`go` is a stack station: it pins no ingredient, declares no stage, worker or `vcs`, and holds no template and no
recipe. Applied, it adds entries to `.cook/config.yaml` and `.cook/hooks.yaml` and nothing else, so it adds commands
and a lane to a lifecycle that another station, such as [`agentic-coding`](agentic-coding.md), or the repository itself
sets up. The lanes and the validator act once `modules.verify` is on, and the post-edit hook once `modules.hooks` is
on ([modules](https://github.com/lmarqs/the-cook/blob/main/docs/features/profile.md#modules)).

```yaml
schema: 1
name: go
version: v0.1.0
description: the commands, lane, validator and post-edit hook of a Go module
detect: [go.mod]
entries:
  .cook/config.yaml:
    bootstrap: [go mod download]
    commands:
      lint: gofmt -d {file}
      build: go build -o /dev/null ./...
    verify:
      app_paths: ["**/*.go", "**/go.mod", "**/go.sum", "**/testdata/**"]
      lint_paths: ["**/*.go"]
      lanes:
        unit:
          select: go test {pattern}
          full: go test ./...
      validators:
        - when: [go.mod, go.sum]
          run: go mod tidy -diff
  .cook/hooks.yaml:
    post_edit:
      - paths: ["**/*.go"]
        run: gofmt -w {file}
```

The entries are checked against go 1.26.7. A station holds no version check, because only an ingredient has checks:
`./cook doctor` reports `go` or `gofmt` missing on a machine that lacks them, because a declared command names them.
The validator needs a toolchain whose `go help mod tidy` lists `-diff`.

## Entries

### `.cook/config.yaml`

| Entry | Value | Why |
| --- | --- | --- |
| `bootstrap` | `go mod download` | It fills the module cache in every new workspace, and once in each checkout an attempt runs in, before any worker starts, so a worker's own `go test` finds the modules the build needs without downloading them |
| `commands.lint` | `gofmt -d {file}` | It prints the formatting diff and exits 1 when the file differs from gofmt's formatting, exits 2 with the position on a syntax error, and exits 0 otherwise |
| `commands.build` | `go build -o /dev/null ./...` | It compiles every package of the module and writes nothing: `-o /dev/null` keeps `go build` from writing an executable into the checkout when `./...` matches a single main package |
| `verify.app_paths` | Go sources, every `go.mod` and `go.sum`, every `testdata` directory | A change to any of them can change what builds or what a test sees, so it is an application change: the build runs and every lane is selected or skipped ([lane rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/verify.md#lane-rules)) |
| `verify.lint_paths` | `**/*.go` | Every changed Go file goes through `commands.lint` |
| `verify.lanes.unit` | `go test {pattern}`, and `go test ./...` in full | `{pattern}` is a package pattern, such as `./internal/billing/...`. `go test` also runs a high-confidence subset of the `go vet` checks (`atomic`, `bool`, `buildtags`, `directive`, `errorsas`, `ifaceassert`, `nilfunc`, `printf`, `stringintconv`, `tests`) and fails the package when one reports a problem, so the station declares no separate vet lane |
| `verify.validators` | `go mod tidy -diff`, when `go.mod` or `go.sum` changes | It prints the changes `go mod tidy` would make, without writing them, and exits non-zero when there are any, so a module file left untidy fails `verify` |

`go test ./...` run in the primary checkout never descends into the managed workspaces under `.work/workspaces/`:
the go tool ignores directories whose names begin with `.` or `_`, and directories named `testdata`.

### `.cook/hooks.yaml`

The post-edit hook runs `gofmt -w {file}` on each `.go` file a worker edits: gofmt rewrites the file in its own
formatting, so the worker sees only what needs judgement. On a syntax error gofmt exits 2 and prints the position, and
`the-cook hook` passes that output to the worker
([post edit](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#post-edit)).

A Go repository that commits generated code keeps the generator's output out of the hook, and runs its validators on
edits too:

```yaml
# .cook/hooks.yaml
schema: 1
post_edit:
  - paths: ["**/*.go"]
    exclude: ["**/*_gen.go"]
    run: gofmt -w {file}
validators: { from: verify.validators }
session_start: { context: true }
raw: {}
```

### Policy, lifecycle actions and standing context

The station adds none of them. Which `go` subcommands a worker runs without a prompt, what an attempt must never run,
how work is delivered and which guidance every session receives are choices each repository makes in its own files.
A Go repository's policy can look like this, with the reason above each group:

```yaml
# .cook/policy.yaml
schema: 1
shared:
  allow:
    # Version control is local or reaches a branch; branch protection guards what lands.
    - [git]
    # The toolchain's local commands. go generate and go run start whatever the code names.
    - [go, [build, test, vet, mod, generate, run]]
    - [golangci-lint, run]
    # Reads of issues and pull requests.
    - [gh, [issue, pr], [list, view]]
    - "mcp__github__get_*"
    - "mcp__github__list_*"
    - "WebFetch(domain:pkg.go.dev)"
  ask: []
  deny:
    # A release is published for everyone and cannot be taken back.
    - [goreleaser, release]
unattended:
  deny:
    # the-cook attempt commits an attempt's work after the replay; the worker never does.
    - [git, [commit, push, worktree]]
    - [gh]
```

This policy holds no `ask` entry for `[the-cook, lifecycle, run, deliver]`, the entry `the-cook scaffold` seeds
([the seeded entry](https://github.com/lmarqs/the-cook/blob/main/docs/features/policy.md#the-seeded-entry)): the
repository's `deliver` actions only push a branch and open a draft pull request, so it leaves that command to each
worker's own default for unlisted commands. The three native rule strings reach only the workers whose ingredients
declare a pattern they match, such as [`claude`](../ingredients/claude.md); the `policy` block of `AGENTS.md` marks
the others.

## Detect rule

`detect: [go.mod]`: a `go.mod` at the root of the repository. `the-cook scaffold --cookbook` applies the station when
it matches, and `the-cook station list --cookbook` shows whether it does
([station detection](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#station-detection)). A
repository whose Go module lives in a subdirectory, such as `cli/go.mod` beside a front end, is not detected: name
the station with `--station go` or apply it by name, then point the commands at that directory.

## Applying it

On a repository without `the-cook`, the scaffold applies `go` when `go.mod` is there, beside the stations named:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --dry-run
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding
```

On a repository that already uses `the-cook`, read the station, then apply it:

```sh
./cook station show go --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply go --cookbook github:lmarqs/the-cookbook@v0.1.0 --dry-run
./cook station apply go --cookbook github:lmarqs/the-cookbook@v0.1.0
git add -A && git commit -m "chore: apply the go station"
```

`station show` prints the `station.yaml` above and its entries grouped by the file they go into. `station apply`
adds only what the files lack and shows it as one diff
([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)). In the diff,
review:

- every `kept` line: a `commands.lint`, `commands.build` or `unit` lane the repository already names stays as it is,
  and the station's differing entry is not written;
- `verify.app_paths`: narrow it when a directory of Go code, such as `examples/`, is not part of what the lanes test;
- the `unit` lane: add `-race` or `-count=1` to the commands when the repository's tests need them;
- the record the profile gains:

```yaml
stations:
  - name: go
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

From then on the entries are the repository's own. A change verifies with the package patterns it touches:

```sh
./cook verify --lane unit=./internal/billing/...
```

Across many repositories, `the-cook fleet` applies it in each and delivers each change through that repository's own
`deliver` actions ([fleet](https://github.com/lmarqs/the-cook/blob/main/docs/features/fleet.md)):

```sh
the-cook fleet station apply go --cookbook github:lmarqs/the-cookbook@v0.1.0 --yes --deliver
```

The station has no renders and no floor, so nothing it wrote changes with a new release of `the-cook` or a new tag of
this cookbook. A newer tag's entries reach the files only when the station is applied again
([applying again](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#applying-again)):

```sh
./cook station apply go --cookbook github:lmarqs/the-cookbook@<tag> --dry-run
./cook station apply go --cookbook github:lmarqs/the-cookbook@<tag>
git diff
```

## Pitfalls

### A package pattern that selects no test passes

- **Symptom.** `./cook verify --lane unit=./internal/billing/...` passes, and no test ran.
- **Why.** `go test` prints `[no test files]` for a package without test files and counts it as passed, and a `-run`
  expression that matches no test prints `testing: warning: no tests to run` and passes too.
- **What the station does.** Nothing beyond the lane rules: an application change selects at least one lane
  ([the-cook pitfalls][zero-tests]).
- **What you do.** Give the lane the packages whose tests exercise the change, and read the lane's output in a
  failing or suspicious run.

### A test result comes from the cache

- **Symptom.** A lane passes in no time and prints `(cached)` after something the tests depend on changed outside the
  module, such as a local database.
- **Why.** In package list mode, which every `{pattern}` of the lane uses, `go test` reuses a passing result when the
  test binary is the same, every flag is a cacheable one, and the files in the module and the environment variables
  the tests read are unchanged. State outside the module is not part of that key.
- **What the station does.** Nothing: for tests that read only the module, the cached result is the right one.
- **What you do.** Add `-count=1` to the lane's commands when the tests read state outside the module.

### A pipeline lint passes a file that does not parse

- **Symptom.** A repository that replaced the lint command with `gofmt -l {file} | (! grep .)` gets a passing
  `verify` for a Go file with a syntax error.
- **Why.** gofmt prints the error on stderr and exits 2, the pipeline's status is that of `! grep .`, and with nothing
  on stdout that status is 0.
- **What the station does.** Lints with `gofmt -d {file}`, whose own exit status is 1 for a formatting difference and
  2 for a syntax error.
- **What you do.** Keep the lint command's own exit status in what `verify` sees.

### A sandboxed worker cannot write the build cache

- **Symptom.** In an attempt, the worker's own `./cook verify` fails building, while the replay of the same change
  passes.
- **Why.** `go build` and `go test` write their build cache to `GOCACHE`, by default a `go-build` directory in the
  user's cache directory (`go env GOCACHE` prints it). A worker whose sandbox opens for writing only the checkout,
  the clone's state directory and its `writable_roots` cannot write there; the replay runs outside the sandbox.
- **What the station does.** Fills the module cache in `bootstrap`, which `the-cook` runs before the worker starts,
  so the worker needs no download.
- **What you do.** Start the loop runner, or the attempts, with `GOCACHE` inside the clone's state directory, which
  an attempt may write: `export GOCACHE="$PWD/.work/go-build"` in the primary checkout, with the `state.dir` the
  profile names in place of `.work`. The go tool's build cache is safe for concurrent use, so every workspace can
  share it.

### An allowed `go generate` or `go run` runs anything

- **Symptom.** A worker allowed `[go, generate]` runs a program no policy entry names.
- **Why.** `go generate` runs the commands that `//go:generate` lines name, and those commands can run any process;
  `go run` builds and runs code the worker can edit. A policy entry matches the command the worker starts, not what it
  runs ([the-cook pitfalls][task-runner]).
- **What the station does.** Allows nothing: it adds no policy entry.
- **What you do.** Allow `generate` and `run` only where running code the worker wrote is acceptable, and keep
  production credentials out of the worker's environment.

### `go -C <dir>` is another spelling of every subcommand

- **Symptom.** `go -C tools test ./...` prompts although `[go, test]` is allowed, and a deny for a `go` subcommand
  does not stop its `-C` spelling.
- **Why.** `-C dir` changes the directory before the command runs and must come first on the command line, so the
  command's leading words are `go -C tools`, which no prefix entry for `go test` matches ([prefix matching][prefix]).
- **What the station does.** Its own commands run from the root of the checkout and never use `-C`.
- **What you do.** Write a prefix entry for each `-C` directory the repository uses, or keep workers on commands run
  from the root, as `AGENTS.md` shows them.

### One lint command and one `unit` lane in a polyglot repository

- **Symptom.** In a repository that applied `go` and then `node`, `verify` lints `web/app.ts` with gofmt and fails, or
  the front end's tests never run.
- **Why.** `commands.lint`, `commands.build` and a lane are named items: the first station applied writes them, and
  the next one's are reported as kept, while the path lists of both are added
  ([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)).
- **What the station does.** Names its lane `unit`, as every stack station of this cookbook does.
- **What you do.** Write one lint command that picks the linter by the file's extension, and declare the second
  stack's lane under a name of its own:

```yaml
commands:
  lint: case {file} in *.go) gofmt -d {file} ;; *) npx --no -- eslint {file} ;; esac
  build: go build -o /dev/null ./... && npm run build --if-present
verify:
  lanes:
    unit:
      select: go test {pattern}
      full: go test ./...
    web:
      select: npm test -- {pattern}
      full: npm test
```

A task runner's task that picks the linter, such as one of [mise](../ingredients/mise.md), does the same job.

[zero-tests]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-lane-passes-with-zero-tests-or-with-unrelated-tests
[task-runner]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-task-runner-runs-whatever-its-tasks-say
[prefix]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-prefix-matches-the-leading-words-only
