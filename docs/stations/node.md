# node

The `node` station gives an npm project the install, lint and build commands, the `unit` lane and the lint-on-edit
hook that `the-cook` runs in each new workspace, at the Verify stage and after each edit a worker makes. Its entries
call `npm`, `npx` and the project's own `package.json` scripts, so they work in a repository that uses no task runner.

## What it sets up

`node` is a stack station: it pins no ingredient, declares no stage, worker or `vcs`, and holds no template and no
recipe. Applied, it adds entries to `.cook/config.yaml` and `.cook/hooks.yaml` and nothing else, so it adds commands
and a lane to a lifecycle that another station, such as [`agentic-coding`](agentic-coding.md), or the repository itself
sets up. The lane acts once `modules.verify` is on, and the post-edit hook once `modules.hooks` is on
([modules](https://github.com/lmarqs/the-cook/blob/main/docs/features/profile.md#modules)).

```yaml
schema: 1
name: node
version: v0.1.0
description: the commands, lane and post-edit hook of an npm project
detect: [package.json]
entries:
  .cook/config.yaml:
    bootstrap: [npm ci]
    commands:
      lint: npx --no -- eslint {file}
      build: npm run build --if-present
    verify:
      app_paths:
        - "**/*.{js,jsx,mjs,cjs,ts,tsx,mts,cts}"
        - "**/package.json"
        - package-lock.json
        - "**/tsconfig*.json"
      lint_paths: ["**/*.{js,jsx,mjs,cjs,ts,tsx,mts,cts}"]
      lanes:
        unit:
          select: npm test -- {pattern}
          full: npm test
  .cook/hooks.yaml:
    post_edit:
      - paths: ["**/*.{js,jsx,mjs,cjs,ts,tsx,mts,cts}"]
        run: npx --no -- eslint --fix {file}
```

The entries are checked against node 22.23.1 and npm 10.9.8, whose `npx` is the same release. The lint command and
the hook run the ESLint the project installs, at whatever version its lock file records. A station holds no version
check, because only an ingredient has checks: `./cook doctor` reports `npm` or `npx` missing on a machine that lacks
them, because a declared command names them.

## Entries

### `.cook/config.yaml`

| Entry | Value | Why |
| --- | --- | --- |
| `bootstrap` | `npm ci` | It removes a `node_modules` already there and installs the versions `package-lock.json` records into the checkout's own, never writes `package.json` or the lock file, and exits with an error when the two disagree. It runs in every new workspace and once in each checkout an attempt runs in, so a nested workspace builds with its own dependencies instead of finding the primary checkout's by walking up directories ([nesting](https://github.com/lmarqs/the-cook/blob/main/docs/features/workspaces.md#nesting)) |
| `commands.lint` | `npx --no -- eslint {file}` | It runs the ESLint the project installs on one file. `--no` refuses to install a package the project lacks, which `npx` otherwise does without asking whenever its standard input is not a terminal, as in a worker's hook or an unattended run |
| `commands.build` | `npm run build --if-present` | It runs the project's `build` script; `--if-present` lets a project without one pass instead of failing every application change |
| `verify.app_paths` | JavaScript and TypeScript sources of every module kind, every `package.json`, `package-lock.json`, every `tsconfig*.json` | A change to any of them can change what builds or what a test sees, so it is an application change: the build runs and every lane is selected or skipped ([lane rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/verify.md#lane-rules)) |
| `verify.lint_paths` | the same sources | Every changed source file goes through `commands.lint` |
| `verify.lanes.unit` | `npm test -- {pattern}`, and `npm test` in full | It runs the `test` script, with the pattern passed to the script after `--`, as npm passes positional arguments. What the pattern means, a file path or a test name, is the test runner's choice |

The station declares no `e2e` lane. npm has a command for the `test` script and none for an end-to-end suite, so any
name the station chose would be a guess, and a declared lane binds every application change to select or skip it. A
repository with an end-to-end suite declares the lane itself ([Applying it](#applying-it)).

### `.cook/hooks.yaml`

The post-edit hook runs `npx --no -- eslint --fix {file}` on each source file a worker edits: ESLint fixes what its
rules can fix in that one file and reports the rest, which `the-cook hook` passes to the worker
([post edit](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#post-edit)).

A Node API that applied the station, narrowed the hook to its source layout, and added a notification and a native
hook of its own:

```yaml
# .cook/hooks.yaml
schema: 1
post_edit:
  - paths: [src/**/*.ts, test/**/*.ts]
    exclude: [src/migrations/**]
    run: npx --no -- eslint --fix {file}
validators: { from: verify.validators }
session_start: { context: true }
attention:
  - name: ping-me
    http:
      url_env: CHAT_WEBHOOK_URL
      json: { text: "$COOK_WORKER needs you in $COOK_REPO ($COOK_BRANCH): $COOK_MESSAGE" }
raw:
  SessionStart: [{ run: bd prime --hook-json }]
```

`CHAT_WEBHOOK_URL` is set in each developer's own shell profile; a developer who does not set it gets an `attention`
failure on stderr, and the session continues. The `raw` entry is the repository's own native hook on `SessionStart`, the
session-start event of [Claude Code](../ingredients/claude.md): it runs `bd prime --hook-json`, which prints the
[Beads](../ingredients/beads.md) workflow context wrapped in the session-start hook envelope, and as a native hook it
runs whether or not the pinned release of `the-cook` is cached
([missing binary](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#the-missing-binary-guard)).

### Policy, lifecycle actions and standing context

The station adds none of them. Which `npm` scripts a worker runs without a prompt, what an attempt must never run, how
work is delivered and which guidance every session receives are choices each repository makes in its own files;
`the-cook`'s policy page shows a Node API's policy
([a Node API](https://github.com/lmarqs/the-cook/blob/main/docs/features/policy.md#a-node-api)). Two entries need more
thought than they look:

- `[npx]` under `allow` lets a worker run any package of the registry, because `npx` runs a command from a local or a
  remote package;
- `[npm, run, test]` under `allow` lets a worker run a test script it just edited ([the-cook pitfalls][task-runner]).

## Detect rule

`detect: [package.json]`: a `package.json` at the root of the repository. `the-cook scaffold --cookbook` applies the
station when it matches, and `the-cook station list --cookbook` shows whether it does
([station detection](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#station-detection)). A
`package.json` that only installs tooling, such as commit hooks or a formatter in a Go repository, matches too: pass
`--no-detect` with a `--station` for each station the repository wants, or delete the station's entries from
`.cook/config.yaml` and `.cook/hooks.yaml`, and its record from `stations:`, after the scaffold.

## Applying it

On a repository without `the-cook`, the scaffold applies `node` when `package.json` is there, beside the stations
named:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --dry-run
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding
```

On a repository that already uses `the-cook`, or one that gained a Node front end beside another stack, read the
station, then apply it:

```sh
./cook station show node --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply node --cookbook github:lmarqs/the-cookbook@v0.1.0 --dry-run
./cook station apply node --cookbook github:lmarqs/the-cookbook@v0.1.0
git add -A && git commit -m "chore: apply the node station"
```

`station apply` adds only what the files lack and shows it as one diff
([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)). In a repository
whose people already wrote their own `build` command, the diff adds the bootstrap, the lint command, the lane, the
path lists and the hook, and reports `kept commands.build: differs from station node`. In the diff, review:

- every `kept` line, such as `kept lanes.unit: differs from station node`: the repository's item stays, and the
  station's is not written. Nothing to do when yours is right; to take the station's, delete yours and apply again;
- the commands against `package.json`: the `test` script must accept a pattern after `--`, and the build script's name
  must be `build`;
- `bootstrap`: a project whose lock file is `yarn.lock` or `pnpm-lock.yaml` replaces `npm ci` and the commands with its
  package manager's ([npm ci needs package-lock.json](#npm-ci-needs-package-lockjson));
- the record the profile gains:

```yaml
stations:
  - name: node
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

A repository with an end-to-end suite adds its lane, and requires it where only that suite proves a change:

```yaml
verify:
  lanes:
    e2e:
      select: npm run test:e2e -- {pattern}
      full: npm run test:e2e
      required_when: ["src/**/*.controller.ts", "src/**/routes/**"]
```

A change then verifies with patterns for each lane, or a reason to skip one:

```sh
./cook verify --lane unit=invoice --skip e2e="no route or controller changed"
```

Across many repositories, `the-cook fleet` applies it in each and delivers each change through that repository's own
`deliver` actions ([fleet](https://github.com/lmarqs/the-cook/blob/main/docs/features/fleet.md)):

```sh
the-cook fleet station apply node --cookbook github:lmarqs/the-cookbook@v0.1.0 --yes --deliver
```

The station has no renders and no floor, so nothing it wrote changes with a new release of `the-cook` or a new tag of
this cookbook; a newer tag's entries reach the files only when the station is applied again
([applying again](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#applying-again)).

## Pitfalls

### `npx` installs a missing package without asking

- **Symptom.** A repository that replaced the lint command with `npx eslint {file}` downloads and runs a package
  from the registry on a machine where the project's dependencies are not installed.
- **Why.** When a requested package is not among the project's dependencies, `npx` installs it into its cache before
  running it. It asks first only on a terminal: it assumes `--yes` whenever its standard input is not a terminal or it
  detects a CI environment, as in a worker's hook or an unattended run.
- **What the station does.** Calls `npx --no -- eslint`, which answers that question with no and fails instead.
- **What you do.** Keep `--no` in every `npx` command of the profile and the hooks file, and run `npm ci` in a checkout
  before its first lint.

### `npm ci` needs `package-lock.json`

- **Symptom.** `./cook workspace create` exits 1 because the bootstrap failed, and the workspace exists without
  `node_modules`.
- **Why.** `npm ci` installs only from `package-lock.json` or `npm-shrinkwrap.json`, and exits with an error when the
  project has neither, or when the lock file and `package.json` disagree.
- **What the station does.** Uses `npm ci`, because it never writes `package.json` or the lock file, so the bootstrap
  cannot leave a change behind in a fresh workspace.
- **What you do.** Commit `package-lock.json`. A project on another package manager replaces the bootstrap and the
  commands with that manager's, and fixes the failed workspace by running the bootstrap in it or removing it.

### `npm ci` runs install scripts with your credentials

- **Symptom.** Creating a workspace runs code nobody reviewed in that change.
- **Why.** `npm ci` runs the lifecycle scripts the `package.json` files declare, `preinstall`, `install`,
  `postinstall` and `prepare` among them, after it installs `node_modules`. `the-cook` runs `bootstrap` itself, before
  any worker starts, with the environment of whoever created the workspace or started the attempt, outside every
  worker's permission checks.
- **What the station does.** Nothing beyond `npm ci`'s own rule: it installs only what the lock file records.
- **What you do.** Review dependency changes as code. A project whose build needs no install script uses
  `npm ci --ignore-scripts`; keep production credentials out of the environment that creates workspaces.

### `--if-present` builds nothing under another name

- **Symptom.** `verify` passes for a change that does not compile.
- **Why.** `npm run build --if-present` exits 0 when `package.json` has no `build` script, so a project whose build
  script is called `compile` or `tsc` builds nothing.
- **What the station does.** Uses `--if-present` so a project without a build passes.
- **What you do.** Point `commands.build` at the project's real build script, without `--if-present`.

### The test script decides what a pattern selects

- **Symptom.** `./cook verify --lane unit=invoice` passes, and no test about invoices ran, or every test ran.
- **Why.** `npm test -- invoice` appends `invoice` to the `test` script, after its `pretest` script and before its
  `posttest` one. Whether the runner the script calls reads it as a file filter, a name filter or nothing, and whether
  it fails when nothing matches, is the runner's behaviour ([the-cook pitfalls][zero-tests]).
- **What the station does.** Passes the pattern as the script's last argument.
- **What you do.** Check what `npm test -- <pattern>` runs in the project, configure the runner to fail when nothing
  matches, and write the lane's `select` with the runner's own filter flag when it needs one.

### Tools in the primary checkout walk into the workspaces

- **Symptom.** In the primary checkout, the full lane, the build or ESLint reports duplicate modules or fails on files
  under `.work/workspaces/`.
- **Why.** Managed workspaces are nested in the primary checkout, and a glob, a test root or a type checker's include
  list that does not skip that directory walks into them ([the-cook pitfalls][nested-globs]).
- **What the station does.** Nothing: it never edits the project's tool configuration.
- **What you do.** Exclude `.work/**`, or the `state.dir` the profile names, in the test runner, the type checker's
  configuration and ESLint's ignores.

### One lint command and one `unit` lane in a polyglot repository

- **Symptom.** In a repository that applied another stack station first, `verify` lints `.ts` files with that stack's
  linter, or the Node tests never run.
- **Why.** `commands.lint`, `commands.build` and a lane are named items: the first station applied writes them, and the
  next one's are reported as kept, while the path lists of both are added.
- **What the station does.** Names its lane `unit`, as every stack station of this cookbook does.
- **What you do.** Write one lint command that picks the linter by the file's extension, and declare the Node lane
  under a name of its own, as the [go station shows](go.md#one-lint-command-and-one-unit-lane-in-a-polyglot-repository).

[zero-tests]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-lane-passes-with-zero-tests-or-with-unrelated-tests
[task-runner]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-task-runner-runs-whatever-its-tasks-say
[nested-globs]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#the-repositorys-own-globs-pick-up-nested-workspaces
