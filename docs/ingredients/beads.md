# beads

The `beads` ingredient describes [Beads](https://github.com/steveyegge/beads), a task tracker kept in the clone and
driven by its `bd` command, so that `./cook doctor` checks the version of `bd` a repository's tracker actions run, and
`./cook ingredient list --cookbook` shows a repository that uses Beads. It fills no contract and renders nothing:
`the-cook` reaches the tracker only through lifecycle actions, which the
[agentic-coding station](../stations/agentic-coding.md#entries) writes as the repository's own entries.

## What it describes

| | |
| --- | --- |
| Program | Beads, used through its `bd` command; a task is a bead, and an epic's children have ids such as `acme-api-a10.1` |
| Versions | `>=1.3 <1.4`; the facts on this page were checked against bd 1.3.1 |
| Contract | none |
| Detect | `.beads/` |

`bd init` creates `.beads/` and a Dolt database in it: by default an embedded one, in `.beads/embeddeddolt/`, with a
single writer. Machines share a database through `bd dolt push` and `bd dolt pull`, against `refs/dolt/data` on the git
remote; `.beads/issues.jsonl` is an export for viewers, not the source of truth. `bd` keeps a `.beads/.gitignore` of
its own. Nothing in `the-cook` or in this cookbook initialises, configures or reads a Beads database: creating one is
`bd`'s job. `the-cook remove` touches neither `.beads/` nor `bd`, which keep working without it.

A tracker reaches the lifecycle through actions on three events, scoped to `task`, which run `bd` with the task id from
`COOK_TASK` ([trackers](https://github.com/lmarqs/the-cook/blob/main/docs/features/tasks.md#trackers)). The
[agentic-coding station](../stations/agentic-coding.md#entries) writes these:

| Event | Action | `bd` command |
| --- | --- | --- |
| `start` | `claim-task` | `bd update "$COOK_TASK" --status in_progress --assignee "$COOK_ACTOR"` |
| `completed` | `close-task` | `bd close "$COOK_TASK" --reason "verified on $COOK_BRANCH"` |
| `blocked` | `block-task` | `bd update "$COOK_TASK" --status blocked --append-notes "blocked: ${COOK_REASON:-no reason given}"` |

Each command is prefixed with `BEADS_DIR="$COOK_REPO/.beads"`. The facts of `bd` they rely on:

- `bd ready`, which ralph-tui's Beads plugin asks for the next task, lists open beads with no active blocker and leaves
  out `in_progress`, `blocked` and `deferred` ones. A bead the `blocked` action marks stays out of every later run until
  a person reopens it.
- `bd update` and `bd close` fall back to the last touched bead only when no id is given and stdin is a terminal; the
  actions always name the bead.
- `BEADS_DIR` names the `.beads` directory itself, and an explicit one is authoritative when `bd` looks for its
  database, so every action writes the primary checkout's database whichever checkout fires the event.
- `bd` records the actor of each change, from `--actor`, `$BEADS_ACTOR`, git's `user.name` or `$USER`. The
  assignee the `start` action writes is the workspace's actor, `<user>:<workspace>:<random>`.

## Checks

| Check | Value |
| --- | --- |
| `version` | `bd --version`, which prints a line such as `bd version 1.3.1 (Homebrew)`; supported: `>=1.3 <1.4` |
| `detect` | `.beads/` |

`./cook doctor` also reports `bd` missing from `PATH` on a machine where a declared action names it.

## Pitfalls

### Denying `bd close` does not stop a bead closing

- **Symptom.** A bead closes from inside an attempt although the policy denies `bd close`.
- **Why.** `bd done` is an alias of `bd close`, and `bd update --status closed` closes a bead as well.
- **What the ingredient does.** Nothing: the ingredient holds no floor, because what a worker may do with the tracker
  is each repository's choice. The [agentic-coding station](../stations/agentic-coding.md#entries) denies `[bd]` as a
  whole to unattended runs, and its workers do not get `.beads/` as a writable root.

### A person's claim keeps a bead

- **Symptom.** The `start` action fails for a bead a person claimed, and the run goes on.
- **Why.** `bd update` refuses to give another assignee a bead under another actor's live `in_progress` claim unless
  `--force` is given.
- **What the ingredient does.** Nothing. A failed automatic action is reported and never changes the exit code of the
  command that fired it; release the claim with `bd unclaim <id>` before the run, or leave it.

### A workspace can find a database of its own first

- **Symptom.** An action updates a stale copy of the tracker.
- **Why.** When a team tracks `.beads/` in git, every managed workspace checks out its own copy, and `bd` looks for
  `.beads/` in the current directory before it resolves a git worktree to its main checkout.
- **What the ingredient does.** Nothing. The station's actions set `BEADS_DIR="$COOK_REPO/.beads"`, the database of
  the primary checkout; an action written by hand needs the same.

### `bd create --parent` copies the parent's labels

- **Symptom.** Child beads carry labels meant for the epic.
- **Why.** `bd create --parent` copies the parent's labels unless it is given `--no-inherit-labels`.
- **What the ingredient does.** Nothing: creating beads is `bd`'s. Pass `--no-inherit-labels`.

### A dependency on an epic blocks every child

- **Symptom.** No bead of an epic is ready, although none of them depends on anything.
- **Why.** `bd ready` treats every descendant of a blocked bead as blocked, so a dependency put on the epic holds all
  its children.
- **What the ingredient does.** Nothing: readiness is `bd`'s. Put each dependency on the story that needs it.

### `bd init` installs git hooks and agent instructions

- **Symptom.** After a hand-run `bd init`, commits run Beads hooks, and `AGENTS.md` gains a Beads section.
- **Why.** `bd init` installs git hooks, and writes `AGENTS.md` and Claude Code, Codex and Cursor setup, unless it is
  given `--skip-hooks` and `--skip-agents`.
- **What the ingredient does.** Nothing: neither `the-cook` nor this cookbook runs `bd init`. Run
  `bd init --skip-hooks --skip-agents` when the repository keeps its own hooks and agent instructions; a section `bd`
  adds to `AGENTS.md` outside the managed blocks stays as it is.

### A shared database shows claims on other machines

- **Symptom.** A teammate's machine shows beads assigned to an actor such as `ana:invoice-rounding:5c1e`.
- **Why.** A database pushed to a remote carries its assignments and notes, those the lifecycle actions write
  included.
- **What the ingredient does.** Nothing: it neither creates nor configures the database. Run unattended work against a
  database nobody else pulls, or accept that claims are shared; deny `bd dolt push` to workers in the policy when
  sharing must stay a person's step.

## Examples

### Pinning it

```sh
./cook ingredient pin beads --cookbook github:lmarqs/the-cookbook@v0.1.0
```

```yaml
# .cook/config.yaml
ingredients:
  - name: beads
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
```

The ingredient declares no worker and writes no `vcs:`. The
[agentic-coding station](../stations/agentic-coding.md) pins it with the actions above.

### Tracker actions without the station

A repository that wires Beads by hand declares the actions in its own `.cook/hooks.yaml`:

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
```

`completed` with scope `task` is refused without a valid receipt, so the bead closes only after its checks pass
([the `completed` guard](https://github.com/lmarqs/the-cook/blob/main/docs/features/tasks.md#the-completed-guard)).
Linking a bead to an issue of the hosting platform, with `bd update <id> --external-ref <ref>`, stays a person's or a
worker's step.

### The Beads workflow at session start

`bd prime --hook-json` prints Beads' workflow text wrapped for a session-start hook. Declared as a `raw` hook, it runs
as written for every worker whose native events include the one it names, attempts included:

```yaml
# .cook/hooks.yaml
raw:
  SessionStart: [{ run: bd prime --hook-json }]
```

The text is Beads' command reference for agents. In an attempt, where the station denies `bd`, every command it
suggests meets the deny, so keep the hook to repositories whose workers do not run unattended, or replace the text with
a `.beads/PRIME.md` of your own.
