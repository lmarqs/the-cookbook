# terraform

The `terraform` station gives a Terraform configuration the format check, the `validate` and `plan` lanes and the
format-on-edit hook that `the-cook` runs at the Verify stage and after each edit a worker makes. Its entries call the
`terraform` command itself, so they work in a repository that uses no task runner.

## What it sets up

`terraform` is a stack station: it pins no ingredient, declares no stage, worker or `vcs`, and holds no template and
no recipe. Applied, it adds entries to `.cook/config.yaml` and `.cook/hooks.yaml` and nothing else, so it adds commands
and lanes to a lifecycle that another station, such as [`agentic-coding`](agentic-coding.md), or the repository itself
sets up. The lanes act once `modules.verify` is on, and the post-edit hook once `modules.hooks` is on
([modules](https://github.com/lmarqs/the-cook/blob/main/docs/features/profile.md#modules)).

```yaml
schema: 1
name: terraform
version: v0.1.0
description: the format command, validate and plan lanes and post-edit hook of a Terraform configuration
detect: ["**/*.tf"]
entries:
  .cook/config.yaml:
    commands:
      lint: terraform fmt -check -diff -no-color {file}
    verify:
      app_paths:
        - "**/*.tf"
        - "**/*.tf.json"
        - "**/*.tfvars"
        - "**/*.tfvars.json"
        - "**/*.tftest.hcl"
        - "**/.terraform.lock.hcl"
      lint_paths: ["**/*.tf", "**/*.tfvars", "**/*.tftest.hcl"]
      lanes:
        validate:
          select: >-
            terraform -chdir={pattern} init -backend=false -input=false -no-color > /dev/null &&
            terraform -chdir={pattern} validate -no-color
        plan:
          select: >-
            terraform -chdir={pattern} init -input=false -no-color > /dev/null &&
            terraform -chdir={pattern} plan -input=false -lock=false -no-color
  .cook/hooks.yaml:
    post_edit:
      - paths: ["**/*.tf", "**/*.tfvars", "**/*.tftest.hcl"]
        run: terraform fmt -no-color {file}
```

The entries are checked against Terraform 1.13.5. A station holds no version check, because only an ingredient has
checks: `./cook doctor` reports `terraform` missing on a machine that lacks it, because a declared command names it.

## Entries

### `.cook/config.yaml`

| Entry | Value | Why |
| --- | --- | --- |
| `commands.lint` | `terraform fmt -check -diff -no-color {file}` | It prints the formatting diff and exits non-zero when the file differs from the canonical format, exits 2 on a syntax error, and exits 0 otherwise. `fmt` handles `.tf`, `.tfvars` and `.tftest.hcl` files and leaves the JSON forms alone |
| `verify.app_paths` | configuration and variable files in both syntaxes, test files, every `.terraform.lock.hcl` | A change to any of them changes what Terraform would do, so it is an application change and every lane is selected or skipped ([lane rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/verify.md#lane-rules)) |
| `verify.lint_paths` | the native-syntax files `fmt` formats | Every changed one goes through `commands.lint` |
| `verify.lanes.validate` | `init -backend=false`, then `validate`, in the directory `{pattern}` names | The pattern is a root module's directory, such as `envs/staging`, or `.` for the repository root. `validate` reads only the configuration and reaches no remote service, but needs an initialized directory with the providers and modules installed, which `init -backend=false` gives without reaching the backend. The lane drops `init`'s progress on stdout and keeps its errors, which Terraform writes to stderr |
| `verify.lanes.plan` | `init`, then `plan -input=false -lock=false`, in the directory `{pattern}` names | `plan` shows what an apply would change and performs none of it. `-input=false` fails on a missing variable instead of waiting for an answer nobody gives, and `-lock=false` takes no state lock, so an interrupted lane leaves no lock behind to block the team's next apply |

Every command passes `-no-color`, because Terraform writes colour codes into its messages even when they do not go to
a terminal, and a worker would read them as text. The station declares no `bootstrap`, no `commands.build`, no `full`
command and no `required_when`: the root modules, which directories a full run would cover and which changes need a
plan are each repository's own, so each lane initializes the directory its pattern names itself. `./cook verify --full`
therefore still asks for `--lane validate=<dir>` and `--lane plan=<dir>`, or a reason to skip one.

The station declares no `unit` lane either: `terraform test` creates real infrastructure, as its help says, so a lane
that runs it is not a unit lane ([Pitfalls](#terraform-test-creates-real-infrastructure)).

### `.cook/hooks.yaml`

The post-edit hook runs `terraform fmt -no-color {file}` on each `.tf`, `.tfvars` or `.tftest.hcl` file a worker
edits: `fmt` rewrites the file in the canonical format and lists it, and on a syntax error it exits 2 with the
position, which `the-cook hook` passes to the worker
([post edit](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#post-edit)).

A Terraform repository that keeps a check of its own for module variables, and a guard on its state files, declares
them beside the station's hook:

```yaml
# .cook/hooks.yaml
schema: 1
post_edit:
  - paths: ["**/*.tf", "**/*.tfvars", "**/*.tftest.hcl"]
    run: terraform fmt -no-color {file}
validators:
  - when: [modules/**/variables.tf]
    run: ./scripts/check-variable-docs.sh
session_start: { context: true }
raw:
  PreToolUse: [{ matcher: Edit|Write, run: ./scripts/protect-state-files.sh }]
```

The path guard is a hook `the-cook` does not model, a check before a tool runs, so it is a `raw` entry. `PreToolUse` and
the `Edit|Write` matcher are [Claude Code](../ingredients/claude.md)'s native event and tool names; the entry is
rendered only for the workers whose program has that event, and as a native hook it runs whether or not the pinned
release of `the-cook` is cached
([hooks](https://github.com/lmarqs/the-cook/blob/main/docs/features/hooks.md#reference)).

### Policy, lifecycle actions and standing context

The station adds none of them. What a worker may run against real infrastructure, what an attempt must never run, how
work is delivered and which guidance every session receives are choices each repository makes in its own files
([placing a policy entry](https://github.com/lmarqs/the-cook/blob/main/docs/features/safety.md#placing-a-policy-entry)).
For a Terraform repository the usual placement is:

| Decision | Examples |
| --- | --- |
| `deny` | `terraform apply` |
| unlisted | `kubectl apply`, an `aws` command that writes, `gh repo edit` |

Production stays unlisted rather than denied because a `deny` would also stop the person who means it, such as a
developer who asks the agent to run a planned `kubectl apply`. A Terraform repository's policy can look like this,
with the reason above each group:

```yaml
# .cook/policy.yaml
schema: 1
shared:
  allow:
    # Version control is local or reaches a branch; branch protection guards what lands.
    - [git]
    # Commands that change no infrastructure and no state.
    - [terraform, [fmt, validate, providers, version]]
    - [terraform, plan]
    - [tflint]
  ask:
    # Delivery runs the repository's lifecycle actions, outside the worker's own permission checks.
    - [the-cook, lifecycle, run, deliver]
  deny:
    # Commands that change real infrastructure or the state that records it.
    - [terraform, [apply, destroy, import, refresh, taint, test, untaint]]
    - [terraform, state, [mv, rm, push, replace-provider]]
unattended:
  deny:
    # An attempt reaches no hosting platform and no cloud, and never takes newer provider versions.
    - [gh]
    - [aws]
    - [terraform, init, "-upgrade"]
```

`refresh` updates the state to match the remote systems, and `test` creates real infrastructure, so both sit with
`apply`. `init -upgrade` installs the newest versions the constraints allow instead of the ones the lock file records.
None of these entries stops the `-chdir` spelling of the same command
([Pitfalls](#terraform--chdirdir-apply-passes-a-terraform-apply-deny)).

## Detect rule

`detect: ["**/*.tf"]`: a `.tf` file anywhere in the repository, at its root included. `the-cook scaffold --cookbook`
applies the station when it matches, and `the-cook station list --cookbook` shows whether it does
([station detection](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#station-detection)). A
repository that holds `.tf` files it never runs, such as fixtures of another tool's tests, matches too: pass
`--no-detect` with a `--station` for each station the repository wants.

## Applying it

On a repository without `the-cook`, the scaffold applies `terraform` when a `.tf` file is there, beside the stations
named:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --dry-run
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding
```

On a repository that already uses `the-cook`, read the station, then apply it:

```sh
./cook station show terraform --cookbook github:lmarqs/the-cookbook@v0.1.0
./cook station apply terraform --cookbook github:lmarqs/the-cookbook@v0.1.0 --dry-run
./cook station apply terraform --cookbook github:lmarqs/the-cookbook@v0.1.0
git add -A && git commit -m "chore: apply the terraform station"
```

`station apply` adds only what the files lack and shows it as one diff
([merge rules](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#merge-rules)). In the diff,
review:

- every `kept` line: a `commands.lint` or a `validate` or `plan` lane the repository already names stays as it is;
- `.gitignore`: it must ignore `.terraform/`, which every lane's `init` creates
  ([init changes the tree](#init-changes-the-tree));
- the `plan` lane: give it `required_when` for the directories only a plan can prove, such as `modules/**`, knowing
  every run that changes them then needs the backend and its credentials, unattended runs included;
- `bootstrap`: a repository whose workers run in a sandbox without network access initializes each root module there,
  such as `terraform -chdir=envs/staging init -backend=false -input=false`, which `the-cook` runs before any worker
  starts;
- the record the profile gains:

```yaml
stations:
  - name: terraform
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

A change then verifies with the root modules it touches:

```sh
./cook verify --lane validate=envs/staging --lane plan=envs/staging
./cook verify --lane validate=envs/staging --skip plan="only comments and formatting changed"
```

Across many repositories, `the-cook fleet` applies it in each and delivers each change through that repository's own
`deliver` actions ([fleet](https://github.com/lmarqs/the-cook/blob/main/docs/features/fleet.md)):

```sh
the-cook fleet station apply terraform --cookbook github:lmarqs/the-cookbook@v0.1.0 --yes --deliver
```

The station has no renders and no floor, so nothing it wrote changes with a new release of `the-cook` or a new tag of
this cookbook; a newer tag's entries reach the files only when the station is applied again
([applying again](https://github.com/lmarqs/the-cook/blob/main/docs/features/stations.md#applying-again)).

## Pitfalls

### `terraform -chdir=<dir> apply` passes a `[terraform, apply]` deny

- **Symptom.** A worker runs `terraform -chdir=envs/prod apply` in a repository whose policy denies
  `[terraform, apply]`.
- **Why.** `-chdir` is a global option, written before the subcommand, so the command's leading words are
  `terraform -chdir=envs/prod apply`, which no prefix entry for `terraform apply` matches ([prefix matching][prefix]).
  The station's own lanes use that spelling, so a worker that reads them learns it.
- **What the station does.** Nothing: it adds no policy entry.
- **What you do.** Add bare strings such as `"terraform -chdir=* apply*"` and `"terraform -chdir=* destroy*"` to
  `deny`, which the workers whose program matches whole commands by glob enforce; the `policy` block of `AGENTS.md`
  marks the others. Keep credentials that can apply out of the environment of every worker that cannot enforce them,
  and of every unattended run.

### `terraform test` creates real infrastructure

- **Symptom.** A lane named `unit` that runs `terraform test` creates resources in a cloud account.
- **Why.** `terraform test` runs the `run` blocks of the `.tftest.hcl` files and checks their assertions against the
  infrastructure it created, then tries to destroy it. A lane runs as `the-cook` runs it, with the credentials of
  whoever runs `verify`, the replay of an attempt included.
- **What the station does.** Declares no lane that runs `terraform test`.
- **What you do.** Run `terraform test` only where creating and destroying those resources is acceptable, and keep it
  out of the lanes unattended runs select.

### `init` changes the tree

- **Symptom.** `verify` fails because files changed while it ran, or an attempt refuses to start because the
  workspace has uncommitted changes.
- **Why.** `init` writes the local data Terraform needs, the providers and modules it installs included, into a
  `.terraform/` directory in the root module's directory, and when the configuration has no `.terraform.lock.hcl` it
  creates one to record the provider selections it made. A file no ignore rule covers is a new untracked file.
- **What the station does.** Runs `init` inside each lane, and lists `.terraform.lock.hcl` in `app_paths` so a lock
  file change is an application change.
- **What you do.** Ignore `.terraform/` in `.gitignore`, and commit each root module's `.terraform.lock.hcl` once,
  from a run of `init`.

### A plan runs with the credentials of whoever verifies

- **Symptom.** The worker's own `./cook verify --lane plan=<dir>` fails in an attempt, while the replay of the same
  change passes, or the other way round.
- **Why.** The plan lane initializes the backend and reads the state and the remote objects, so it needs network
  access and the credentials in its environment. The worker's `verify` runs inside the worker's sandbox and
  environment, the replay in the environment of the process that started the attempt. `init` installs the provider
  plugins the configuration requires, and `plan` runs them, so a worker that edits the configuration chooses code that
  runs with those credentials.
- **What the station does.** Leaves `plan` without `required_when`, so an attempt can verify with `validate` alone.
- **What you do.** Give the environment that runs plans read-only credentials, and require `plan` only where a person
  runs the change, or where an unattended run's environment is meant to hold them.

### `plan -detailed-exitcode` fails a lane that has changes

- **Symptom.** A repository adds `-detailed-exitcode` to the plan lane, and every change that alters infrastructure
  fails `verify`.
- **Why.** With `-detailed-exitcode`, `plan` exits 2 when it succeeds with a diff, and a lane treats every non-zero
  exit as a failure.
- **What the station does.** Runs `plan` without it, so a plan that succeeds passes, with or without changes.
- **What you do.** Read the plan's output in the lane's log; keep `-detailed-exitcode` for checks that must find no
  changes, such as a drift check, outside the lanes.

### One lint command in a polyglot repository

- **Symptom.** In a repository that applied another stack station first, `verify` lints `.tf` files with that stack's
  linter, or the Terraform files are never linted.
- **Why.** `commands.lint` is a named item: the first station applied writes it, and the next one's is reported as
  kept, while the path lists of both are added. The `validate` and `plan` lanes have names no other stack station of
  this cookbook uses, so they are always added.
- **What the station does.** Nothing beyond the merge rules.
- **What you do.** Write one lint command that picks the linter by the file's extension, as the
  [go station shows](go.md#one-lint-command-and-one-unit-lane-in-a-polyglot-repository).

[prefix]: https://github.com/lmarqs/the-cook/blob/main/docs/features/pitfalls.md#a-prefix-matches-the-leading-words-only
