# git

The `git` ingredient lets `the-cook` run its lifecycle on a git repository: it fills the version-control contract with
git's commands, renders the ignore rules for the state paths, and holds the floor rule that keeps every worker from
force-pushing the default branch.

## What it describes

- **Program.** [git](https://git-scm.com), from 2.23 up to, not including, 3.0. 2.23 is the first release with
  `git switch`, which the `switch` operation runs; `git branch --show-current`, which `branch` runs, is in git from
  2.22. The commands below were checked against git 2.43.0.
- **Contract.** The version-control
  [contract](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-version-control-contract),
  version 1. The profile names the ingredient with `vcs: git`.
- **Detect.** `.git`, which is a directory in a clone and a file in a linked worktree.
- **Remote.** The operations that read or update the remote use the remote named `origin`.

`the-cook` contains no code for git: every git command it runs is in this ingredient's operations. Every command that
reads the repository's history needs `git` on the machine; without it such a command exits 3 naming `git`, and
`doctor` reports the program missing. No operation writes git configuration, installs a git hook or changes
`core.hooksPath`, so Husky and other hook managers keep working as they are.

## Renders

Two convention blocks of `.gitignore`, one per
[state path](https://github.com/lmarqs/the-cook/blob/main/docs/features/profile.md#state-paths):

| Block | Template | With the default paths |
| --- | --- | --- |
| `state` | `/{{state_dir}}/` | `/.work/` |
| `workspaces` | `/{{workspaces_dir}}/` | `/.work/workspaces/` |

```gitignore
# cook:state
/.work/
# /cook:state

# cook:workspaces
/.work/workspaces/
# /cook:workspaces
```

The markers are those of a
[managed block](https://github.com/lmarqs/the-cook/blob/main/docs/features/render.md#managed-blocks) in a hash-comment
file; every other line of `.gitignore` stays as people wrote it. A block leaves out a path the repository tracks, which
the binary asks through the `tracked` operation. Each block lists its own path even when the other already covers it, so
deleting one block never un-ignores a path the other still needs. Both are convention blocks: they say where state and
workspaces live, an eject keeps their text, and a person or a worker who creates a worktree by hand with
`git worktree add .work/workspaces/<name>` follows the same convention. What the state directory holds, the receipt, the
session and an attempt's records included, stays out of every commit.

The worker ingredients render the
[hook command](https://github.com/lmarqs/the-cook/blob/main/docs/features/render.md#the-hook-command) from this
ingredient's `root` operation. With `git` it is:

```sh
"$(git rev-parse --show-toplevel)/cook" --no-download hook <event>
```

It finds the wrapper at the root of the checkout the worker works in, the primary checkout or a workspace, and never
downloads.

## Operations

The table maps each operation of the contract to the command that fills it. A command that needs a shell runs `sh -c`
with its values as positional parameters, so no value reaches the shell's parser.

| Operation | Runs | The binary uses it for |
| --- | --- | --- |
| `head` | `git rev-parse --verify HEAD` | the base an attempt records; a workspace's default base |
| `root` | `git rev-parse --show-toplevel` | the hook command |
| `primary_root` | `git rev-parse --git-common-dir`, then the absolute path of its parent, through `sh` | the primary checkout from a workspace: `COOK_REPO` and the state shared by every workspace |
| `branch` | `git branch --show-current` | `COOK_BRANCH`; the `deliver` guard, which refuses the default branch and no branch |
| `default_branch` | the branch `refs/remotes/origin/HEAD` points to, without its prefix, through `sh` | `COOK_DEFAULT_BRANCH`; `{default_branch}` in the floor; the base of `verify` outside a workspace and of a recipe run; the default branch `scaffold` reports |
| `show` | `git cat-file blob {rev}:{path}` | reading the profile, the policy, the hooks file and the standing context from an attempt's base; the tamper check |
| `clean` | `git status --porcelain --untracked-files=normal`, true when it prints nothing, through `sh` | an attempt refuses uncommitted changes; the `deliver` guard; `workspace remove`; a fleet refuses a dirty repository |
| `tracked` | `git --literal-pathspecs ls-files --error-unmatch -- {path}` | the paths the ignore blocks leave out |
| `changed` | `git diff --name-only --no-renames {rev} --`, then `git ls-files --others --exclude-standard`, both with `core.quotePath=false`, through `sh` | the changed files of `verify` |
| `patch` | `git add --all --intent-to-add`, then `git diff --binary {rev} --` with fixed output options, through `sh` | `last-attempt.patch` of a failed attempt |
| `merge_base` | `git merge-base {a} {b}` | the base of `verify` outside a workspace |
| `landed` | `git cherry {target} {branch}`, keeping the commits marked `+`, through `sh` | `landed` in `workspace remove`; the landed branches `doctor` lists |
| `reset` | `git reset --hard {rev}`, then `git clean -fd`, through `sh` | returning a failed attempt's checkout to its base |
| `commit` | `git add --all`, then `git commit -m {message}`, through `sh` | `--commit` of `the-cook attempt` and `the-cook make`; a fleet delivery |
| `update` | `git fetch origin` | a fleet delivery, before its branch |
| `branch_create` | `git branch {branch} {rev}` | a fleet delivery's branch |
| `switch` | `git switch {branch}` | a fleet delivery's branch |
| `workspace_add` | `git -c core.hooksPath=/dev/null worktree add -b {branch} {path} {rev}` | `workspace create` and recipe runs: a linked worktree on `work/<name>` |
| `workspace_remove` | `git worktree remove {path}` | `workspace remove` |

What the forms guard against, each checked against git 2.43.0:

- **Exit status.** A pipeline exits with the status of its last command, and `test` or `cd` with their own, whatever the
  command substitution in their arguments did, so a git failure would read as an answer:
  `test -z "$(git status --porcelain)"` is true outside a repository, and `git cherry main x | sed ...` reports every
  commit as landed when `main` does not exist. `clean`, `landed`, `primary_root` and `default_branch` assign git's
  output first and stop on its failure.
- **`default_branch`.** It reads `refs/remotes/origin/HEAD`, which `git clone` records. When `origin` exists and that
  ref does not, it fails with `run git remote set-head origin --auto`. In a repository with no `origin` remote, it
  reports the local branch `init.defaultBranch` names, when that branch exists.
- **`show`** runs the plumbing `git cat-file blob`, which prints a file's bytes as committed, never converted for
  display.
- **`tracked`** passes the path with `--literal-pathspecs`, so a path with `*` or `[` is not read as a glob that
  matches other tracked files.
- **`changed`** passes `--no-renames`, so a renamed file lists its old path as well as its new one, and
  `core.quotePath=false`, so a name with bytes above 0x80, such as UTF-8, is printed verbatim instead of quoted.
- **`patch`** marks every untracked file that is not ignored with `--intent-to-add`, so `git diff {rev}` shows it as a
  new file, and passes `--no-color --no-ext-diff --no-textconv --src-prefix=a/ --dst-prefix=b/`, so no personal
  setting, external diff program or text conversion turns the patch into one `git apply` cannot apply. The
  intent-to-add entries stay in the index until a reset clears them; after a failed attempt, `reset` follows at once.
- **`reset`** removes untracked files with `git clean -fd`, which leaves ignored files, so the state directory and
  the `last-attempt.patch` saved there stay.
- **`workspace_add`** sets `core.hooksPath=/dev/null` for that one command, so the repository's `post-checkout` hook,
  which git runs after `git worktree add`, does not run while the workspace is created. The repository's `bootstrap`
  prepares the workspace instead.
- **`workspace_remove`** never forces: git refuses a worktree with uncommitted or untracked files, and a locked one.

## Floor

The floor denies a force push to the default branch, whatever its name: `main`, `master` or another. `{default_branch}`
is what `default_branch` reports:

```yaml
- [git, push, ["--force", "--force-with-lease", "-f"], origin, ["{default_branch}", "HEAD:{default_branch}"]]
- [git, push, origin, ["+{default_branch}", "+HEAD:{default_branch}"]]
- "git push origin {default_branch} --force*"
- "git push origin {default_branch} -f*"
- "git push origin HEAD:{default_branch} --force*"
- "git push origin HEAD:{default_branch} -f*"
```

The two prefix lists catch the flag-first and the `+` spellings, and render for every worker whose ingredient renders
permissions. The four bare strings catch the flag-last spellings, such as `git push origin main --force`, and render
only for a worker whose program matches whole shell commands by glob: a worker of the [`claude`](claude.md)
ingredient, not one of the [`codex`](codex.md) ingredient, whose rules cannot match in the middle of a command.

As [floor](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-floor), the rules render into
every declared worker's permissions on every render and never into `.cook/policy.yaml`. In a repository whose default
branch is `main`, the `policy` block of `AGENTS.md` lists them in one line:

```markdown
- force pushes to `main`: `git push --force origin main` and its other spellings (*floor rule of the git ingredient*)
```

An entry that would weaken them, such as an `allow` for `git push --force origin main`, is dropped from every worker's
output and reported by `render`, by `doctor` until it is removed, and by `scaffold --adopt`, which leaves it out of
the imported policy. An `allow` of `[git]`, which some programs apply without their own judgement, leaves the floor in
force, because a deny wins over an allow.

The floor holds only while the ingredient is pinned in `ingredients:`.

## Checks

| Check | Value |
| --- | --- |
| `version` | `git --version`, which prints `git version 2.43.0`; the range is `>=2.23 <3.0` |
| `detect` | `.git` |

`doctor` reports a git outside the range: install a supported git, or pin a tag of this cookbook whose `git` supports
it. `ingredient list --cookbook` shows the ingredient as detected in every git repository.

## Pitfalls

### No default branch: `origin/HEAD` is not set

- **Symptom.** `doctor` reports no default branch, so the floor cannot name the branch it protects, and a command that
  needs the default branch stops; the operation prints `refs/remotes/origin/HEAD is not set`.
- **Why.** `git clone` records the remote's default branch as `refs/remotes/origin/HEAD`. A repository whose remote was
  added by hand and then pushed or fetched has no such ref, and no setting of git 2.43 makes `git fetch` create or
  change it, so it also keeps naming a default branch the hosting platform renamed.
- **What the ingredient does.** Reads the default branch from that ref whenever `origin` exists, and names the fix when
  the ref is missing: run `git remote set-head origin --auto`, which asks the remote, then `./cook render`.

### The default branch is the local branch

- **Symptom.** After a `git fetch`, a branch merged on the hosting platform is not reported as landed, and a recipe
  run's workspace or a fleet branch starts from an old commit.
- **Why.** `default_branch` reports a name such as `main`, and git resolves `main` to the local branch
  `refs/heads/main`, which moves only when you update it. `git fetch origin`, the `update` operation, moves
  `origin/main` and leaves `main` where it is.
- **What the ingredient does.** Uses the name as the binary passes it and never moves a local branch. Keep the local
  default branch current where it is checked out, with `git pull --ff-only`.

### A squash merge hides landed work

- **Symptom.** A merged branch is never reported as landed: `the-cook workspace remove` fires no `landed` for it, and
  `doctor` does not list it.
- **Why.** `landed` runs `git cherry`, which matches each commit of the branch by its diff against the commits of the
  target. A squash of several commits is one diff that matches none of them, and a rebase that resolved a conflict
  changes the diff too.
- **What the ingredient does.** Reports a commit as landed only when git finds an equivalent, so nothing unmerged is
  treated as merged. Check the merge, then remove the workspace and delete its branch by hand
  (`git branch -D work/<name>`).

### `git -C` and `git -c` slip past git prefixes

- **Symptom.** `[git, push]` is in `unattended.deny`, yet `git -C ../other push` or
  `git -c core.hooksPath=/dev/null commit` runs.
- **Why.** A prefix rule matches the leading words, and an option before the subcommand changes them: the command
  starts with `git -C`, not `git push`. `-C` also reaches the other checkouts of the clone, and `-c` overrides any
  setting for one command.
- **What the ingredient does.** Nothing beyond its floor; the policy decides. Put `[git, ["-C", "-c"]]` in
  `unattended.deny`, next to the subcommands that leave the workspace ([Examples](#policy-entries-for-git)).

### Force-push spellings the floor does not match

- **Symptom.** A worker force-pushes the default branch although the floor is rendered.
- **Why.** The rules match the spellings above and no others: `git push --force` with no remote or refspec while on
  the default branch, which pushes the current branch to its upstream; another option between `push` and the remote;
  `--force` between the remote and the branch; `--force-with-lease=<ref>`; refspecs such as `main:main`,
  `HEAD:refs/heads/main` and `+refs/heads/main`; `--mirror`; deleting the branch with `--delete` or `:main`; and
  `git -C` or `git -c` before `push`.
- **What the ingredient does.** Denies the common spellings in every worker that renders permissions. Branch
  protection on the hosting platform is the guard for the rest; the floor is defence in depth.

### Your git configuration applies to the commits the binary makes

- **Symptom.** `the-cook attempt --commit` fails after its replay passed, or the commit holds content the replay did
  not check.
- **Why.** `commit` runs `git commit`, which runs the repository's `pre-commit` and `commit-msg` hooks and signs when
  `commit.gpgSign` is set. A hook that fails stops the commit, a hook that rewrites files and stages them commits
  content the replay did not check, and signing in an unattended run needs an agent that already holds the key.
- **What the ingredient does.** Commits with the repository's hooks and your settings, so the checks the repository
  requires of every commit apply. Only `workspace_add` turns hooks off, for that one command; `switch` runs
  `post-checkout`.

### A workspace in a repository with submodules cannot be removed

- **Symptom.** `the-cook workspace remove <name>` fails although the workspace has nothing uncommitted.
- **Why.** `git worktree remove` refuses a worktree that contains submodules unless it is forced.
- **What the ingredient does.** Never forces a removal, so it never deletes uncommitted work or a locked workspace.
  Check the workspace, then run `git worktree remove --force <path>`.

### A workspace name used before cannot be created again

- **Symptom.** `the-cook workspace create <name>`, or a recipe run whose workspace has that name, fails because the
  branch already exists.
- **Why.** `workspace_add` runs `git worktree add -b`, which refuses an existing branch, and `workspace remove` keeps
  `work/<name>`.
- **What the ingredient does.** Never uses `-B`, which would reset the existing branch to the new base. Delete the
  old branch once its work has landed (`git branch -d work/<name>`), or choose another name.

### A workspace relocated by hand

- **Symptom.** `the-cook workspace remove` cannot remove a workspace, and `git worktree list` marks it prunable.
- **Why.** git records the path of each linked worktree in the clone's `.git/worktrees/`, and moving the directory
  leaves that record pointing at the old path.
- **What the ingredient does.** Nothing. Run `git worktree repair` inside the relocated workspace so git finds it again,
  then put it back under the workspaces directory with `git worktree move`, which updates git's record.

### A failed attempt's edits are only in its patch

- **Symptom.** After a failed `the-cook attempt`, the checkout has none of the worker's edits.
- **Why.** The attempt returns the checkout to its base with `git reset --hard` and `git clean -fd`, which also delete
  every untracked file that is not ignored.
- **What the ingredient does.** Its `patch` output, new and binary files included, is saved first as
  `last-attempt.patch` in the checkout's state directory, which `git clean -fd` leaves because the `state` block ignores
  it. Continue by hand with `git apply .work/last-attempt.patch`.

### The base cannot be read

- **Symptom.** An attempt fails before the worker starts because its base cannot be read.
- **Why.** `git rev-parse --verify HEAD` fails: the repository has no commit yet, or git refuses a repository owned by
  another user until `safe.directory` lists it.
- **What the ingredient does.** Reports git's error. Check the checkout with `git status`, then make the first commit,
  or add the directory with `git config --global --add safe.directory <path>`.

### A path with quotes, backslashes or control characters

- **Symptom.** `verify` does not lint a changed file, or a lane does not select it, when its name holds a double quote,
  a backslash, a tab or another control character.
- **Why.** git quotes such a path in its output whatever `core.quotePath` says; that setting stops it from quoting only
  the bytes above 0x80.
- **What the ingredient does.** Sets `core.quotePath=false` for `changed`, so UTF-8 names come through verbatim.
  Rename the other files.

### Ignoring a rendered file breaks the other clones

- **Symptom.** A teammate's clone has no `cook`, or a worker there runs without the repository's permissions.
- **Why.** A rule in `.gitignore` covers a file `the-cook` renders and the repository commits, such as `cook` or a
  worker's settings file, so it never reaches the remote.
- **What the ingredient does.** Ignores only the state paths. Keep `cook`, `.cook/` and every worker's rendered
  settings out of `.gitignore`.

### `cook` loses its executable bit

- **Symptom.** `./cook` fails with permission denied in a fresh clone, and `doctor` reports `cook` is not executable.
- **Why.** git stores the executable bit with each file, and some filesystems lose it, so a commit made from one can
  record `cook` as not executable.
- **What the ingredient does.** Nothing. Run `chmod +x cook`, then `git add --chmod=+x cook`, and commit.

### Delivery refuses the default branch and a detached HEAD

- **Symptom.** `the-cook lifecycle run deliver` exits 3 and runs no action.
- **Why.** The `deliver` guard refuses the default branch and a checkout on no branch; `git branch --show-current`
  prints nothing on a detached HEAD.
- **What the ingredient does.** Reports the branch as git does. Run `./cook workspace create <name>`, or
  `git switch -c <branch>`, then deliver.

## Examples

### Pin it and name it in `vcs:`

```sh
./cook ingredient pin git --cookbook github:lmarqs/the-cookbook@v0.1.0
$EDITOR .cook/config.yaml        # add  vcs: git
./cook render
git diff
```

```yaml
# .cook/config.yaml
ingredients:
  - name: git
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
vcs: git
```

The render adds the `state` and `workspaces` blocks to `.gitignore`, the floor to every declared worker's permissions,
and the floor's line to the `policy` block of `AGENTS.md`. The [`agentic-coding`](../stations/agentic-coding.md)
station pins the ingredient and writes `vcs: git` in the same diff as the rest of the station, and the
[`implement-task`](../recipes/implement-task.md) recipe requires the version-control contract it fills.

### Policy entries for git

Git commands stay in the clone or reach a branch, so interactive workers may run them. Attempts may not: their result is
verified after the worker exits and committed by `the-cook attempt --commit`, and anything that leaves the workspace
before that cannot be checked.

```yaml
# .cook/policy.yaml
schema: 1
shared:
  allow:
    # Local work and branches; branch protection guards what lands on the default branch.
    - [git]
  ask:
    - [the-cook, lifecycle, run, deliver]
  deny: []
unattended:
  deny:
    # Commits, pushes, refs and workspaces leave the attempt before the-cook attempt verifies it.
    - [git, [am, cherry-pick, commit, merge, push, rebase, reset, stash, tag, update-ref, worktree]]
    # Options that act on another checkout or override the repository's configuration.
    - [git, ["-C", "-c"]]
    # The hosting platform's CLI.
    - [gh]
    - [the-cook, workspace]
```

### Delivery and landed actions that run git

```yaml
# .cook/hooks.yaml, the lifecycle section
lifecycle:
  deliver:
    - name: push
      run: git push -u origin "$COOK_BRANCH"
    - name: open-pr
      run: gh pr create --draft --fill --base "$COOK_DEFAULT_BRANCH" --head "$COOK_BRANCH"
      auto: false
  landed:
    - name: delete-remote-branch
      run: git push origin --delete "$COOK_BRANCH"
      auto: true
```

Actions run as child processes of `the-cook`, outside every worker's permissions, so the floor does not apply to
them; the `deliver` guard refuses to run them on the default branch. `landed` fires for a recorded branch whose
commits the `landed` operation finds on the default branch.

### Record the default branch in a clone without it

```sh
git remote set-head origin --auto     # asks the remote, sets refs/remotes/origin/HEAD
./cook render                         # the floor names the branch
./cook doctor
```
