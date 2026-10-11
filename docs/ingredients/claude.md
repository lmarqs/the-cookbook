# claude

The `claude` ingredient makes Claude Code a worker of a repository: it fills the worker contract with Claude Code's
launch, hook payload, instruction file, skills directory and rule syntax, renders the policy and the worker hooks into
`.claude/settings.json`, and refuses the arguments that turn Claude Code's permission checks off.

## What it describes

- **Program.** Claude Code, the `claude` command, from 2.1.292 up to, not including, 2.2. The launch arguments were
  checked against the help of 2.1.292, 2.1.294, 2.1.295 and 2.1.296, and the hook events, payload fields and settings
  keys against 2.1.296.
- **Contract.** The worker
  [contract](https://github.com/lmarqs/the-cook/blob/main/docs/features/ingredients.md#the-worker-contract), version 1.
  A worker uses the ingredient when its `ingredient` is `claude`, or when it is named `claude` and names none.
- **Detect.** `.claude/` and `CLAUDE.md`.
- **Roles.** The launch passes `unattended.deny` as arguments, so a `claude` worker runs attempts in either
  [role](https://github.com/lmarqs/the-cook/blob/main/docs/features/workers.md#roles). Declared `interactive`, people
  use it in their sessions under the shared policy, and each attempt still receives the unattended denies, with
  `[the-cook, lifecycle]` in each spelling `the-cook` renders, at launch. Declared `unattended`, its rendered `deny`
  list carries them too.
- **No sandbox roots.** Claude Code bounds a session by its permission rules, not by a write sandbox opened at launch.
  An attempt writes outside its checkout as far as its permissions let it, and `writable_roots` on a `claude` worker is
  a validation error.

## Renders

| Output | Kind | Holds |
| --- | --- | --- |
| `permissions.allow`, `permissions.ask` and `permissions.deny` of `.claude/settings.json` | owned keys | the policy for the worker's role, with the floor rules of every pinned ingredient in `deny` |
| entries of `hooks.<event>` in `.claude/settings.json` that run the hook command | owned entries | one per worker event the declarations need, with `modules.hooks` on |
| entries of `hooks.<event>` in `.claude/settings.json` from `raw` | owned entries | each `raw` entry of `.cook/hooks.yaml`, with its matcher and its command as written |
| the `import` block of `CLAUDE.md` | managed block | one line: `@AGENTS.md` |
| `.claude/skills/the-cook` | link | `../../.agents/skills/the-cook`, the shipped skill |

They render only for the workers that name the ingredient. Everything else in `.claude/settings.json` is preserved byte
for byte: `env`, `model`, `statusLine`, `enabledPlugins`, `$schema`, the other keys under `permissions` such as
`defaultMode`, and the hook entries a person or a Claude Code plugin added
([merge rendering](https://github.com/lmarqs/the-cook/blob/main/docs/features/render.md#merge-rendering-native-files)).
The ingredient never writes `.claude/settings.local.json`, the user's `~/.claude/settings.json`, `.claude/commands/`,
`.claude/agents/` or `.claude/worktrees/`. A repository that adopted `the-cook` holds:

```text
.claude/
  settings.json             owned keys: the permission lists; owned entries: the-cook's hook entries and raw entries
  settings.local.json       never written; read only by scaffold --import local
  skills/the-cook -> ../../.agents/skills/the-cook
CLAUDE.md                   managed block import
```

### Permissions

Each policy entry renders as Claude Code rule strings, in the list of its decision:

| Policy entry | Rule | Matches in Claude Code |
| --- | --- | --- |
| prefix list `[git, push]` | `Bash(git push *)`, one rule per combination of alternatives | `git push`, and every command that starts with `git push ` |
| native rule string, such as `"WebFetch(domain:pkg.go.dev)"`, `"Read(./.env*)"`, `"WebSearch"` or `"mcp__github__get_*"` | the string verbatim | what Claude Code's own rule syntax matches, globs included |
| bare string, such as `"gh pr merge *--admin*"` | `Bash(gh pr merge *--admin*)` | the whole shell command, with `*` matching anything anywhere |

A string is a native rule string for a `claude` worker when it is a tool name that starts with a capital letter, with
or without a specifier in parentheses, or when it starts with `mcp__`; every other string is a bare string. Claude Code
is the worker in this cookbook that matches whole commands by glob, so bare strings, the flag-last force-push rules of
the [git](git.md#floor) ingredient's floor among them, hold in a `claude` worker, and the `policy` block of
`AGENTS.md` marks them as not enforced in a [`codex`](codex.md) worker. Rules for MCP tools render like any native
rule string; the MCP server definitions, which often carry tokens, stay in each person's own settings.

For this policy:

```yaml
schema: 1
shared:
  allow:
    - [npm, run, [lint, test]]
    - "mcp__github__get_*"
  ask:
    - [the-cook, lifecycle, run, deliver]
  deny:
    - [npm, publish]
unattended:
  deny:
    - [gh]
```

the rendered lists of an `interactive` worker hold these rules, besides the allow rules `the-cook` adds for its own
commands and the floor rules in `deny`:

```json
{
  "permissions": {
    "allow": ["Bash(npm run lint *)", "Bash(npm run test *)", "mcp__github__get_*"],
    "ask": [
      "Bash(the-cook lifecycle run deliver *)",
      "Bash(./cook lifecycle run deliver *)",
      "Bash(cook lifecycle run deliver *)"
    ],
    "deny": ["Bash(npm publish *)"]
  }
}
```

An `unattended` worker's `deny` holds `Bash(gh *)` as well; for an `interactive` one, `[gh]` reaches each attempt as
`--disallowedTools 'Bash(gh *)'` only.

### Hooks

The hook entries are Claude Code's matcher groups, one per native event. With post-edit commands declared:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "\"$(git rev-parse --show-toplevel)/cook\" --no-download hook session-start" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "\"$(git rev-parse --show-toplevel)/cook\" --no-download hook post-tool-use" }
        ]
      }
    ]
  }
}
```

An empty matcher matches every occurrence of the event. The command is the
[hook command](https://github.com/lmarqs/the-cook/blob/main/docs/features/render.md#the-hook-command), with the `root`
operation of the [git](git.md) ingredient. A `raw` entry renders under the event it names, which is one of Claude Code's
own events, such as `PreToolUse` for a path guard before an edit, as in the [terraform](../stations/terraform.md)
station's example; its matcher takes Claude Code's tool names, such as `Edit|Write`.

### Instructions

Claude Code reads `CLAUDE.md`, and loads `AGENTS.md` by itself only in a project that has no `CLAUDE.md`. The `import`
block makes it read `AGENTS.md`, its managed blocks and static context included, whatever else `CLAUDE.md` holds. The
block goes into the root `CLAUDE.md`; a repository without one gets a `CLAUDE.md` that holds only the block. Content
outside the block is never changed: instructions every worker needs belong in `AGENTS.md`, which Codex reads too, and
the ones only Claude Code needs stay in `CLAUDE.md`.

### The shipped skill

Claude Code reads skills from `.claude/skills/<name>/SKILL.md`, so the render links `.claude/skills/the-cook` to
`.agents/skills/the-cook`. When `.claude/skills` is a link to the whole `.agents/skills` directory, the skill is
reached through it and no link is written. When it is a real directory, with skills of its own or links a skills
installer made, the link is added inside it and nothing else there changes.

### Removal

`the-cook remove` deletes the renders. An undo removes the permission lists and writes back, as plain settings, the
rules adoption imported from the project's own files; it removes the `import` block, and deletes a `CLAUDE.md` that held
only the block. An eject keeps the permission lists as the repository's own, minus the rules that name a `the-cook`
command, removes the hook entries that run `cook`, and keeps the `@AGENTS.md` line, so Claude Code still reads
`AGENTS.md` ([eject](https://github.com/lmarqs/the-cook/blob/main/docs/features/remove.md#eject)). Neither touches
`.claude/settings.local.json` or the user's settings.

### Reviewing the diff

After a render, read in `.claude/settings.json` each new `allow` rule and how broad it is, the rules that left `deny`
or `ask`, and new hook commands; in the `policy` block of `AGENTS.md`, the entries marked as not enforced in a worker.
Commit `.claude/settings.json` and never add it to `.gitignore`: teammates' sessions and every workspace read the
committed file.

## Operations

| Contract item | Filled with | What Claude Code does with it |
| --- | --- | --- |
| `launch.command` | `claude --print` | runs one prompt without its terminal interface, prints the response and exits; this mode skips the workspace trust dialog |
| `launch.unattended` | `--permission-mode auto --permission-prompts none` | auto mode, whose classifier judges a command no rule matches; nobody answers a permission prompt, so whatever would prompt is denied |
| `launch.unattended_deny` | `--disallowedTools <rule>`, once per entry, in the rule shape of the permissions | denies each entry of `unattended.deny` for the session |
| `launch.sandbox_roots` | not filled | opens no write sandbox at launch |
| `launch.model` | `--model <model>` | an alias, such as `opus` or `sonnet`, or a model's full name |
| `launch.effort` | `--effort <level>` | `low`, `medium`, `high`, `xhigh` or `max`; another value is ignored with a warning |
| `launch.prompt` | stdin | the prompt of an `attempt` step or action, read by `claude --print` |
| `floor.bypass` | the arguments under [Floor](#floor) | |
| `payload` | the fields under [Hook events](#hook-events) | the hook input, a JSON object on the hook's stdin |
| `instructions` | `CLAUDE.md`, block `import`, line `@AGENTS.md` | [Instructions](#instructions) |
| `skills` | `.claude/skills/the-cook` | [The shipped skill](#the-shipped-skill) |
| `native_rules` | the patterns under [Permissions](#permissions), the files under [Adoption](#adoption) | |

With `unattended.deny` holding `[gh]`, every attempt of a `claude` worker starts with
`--disallowedTools 'Bash(gh *)'`. The arguments a caller passes come after the pinned ones; Claude Code accepts
`--print`, `--model` and `--effort` given twice, so a loop runner that adds its own, as ralph-tui's `claude` plugin adds
`--print`, and `--model` when one is set, does not break the launch.

### Hook events

| Neutral event | Native event | Fires | Payload fields the ingredient maps |
| --- | --- | --- | --- |
| `session-start` | `SessionStart` | at startup, resume, `/clear` and after a compaction, so `always` context is injected again after one | `hook_event_name`, `session_id`, `cwd` |
| `post-tool-use` | `PostToolUse` | after a tool call; the rendered matcher is `Edit\|Write\|MultiEdit`, with `\|Read` added when `when.paths` context exists | `tool_name`, `tool_input.file_path` |
| `prompt-submit` | `UserPromptSubmit` | when the person submits a message | `prompt` |
| `attention` | `Notification` | when Claude Code sends a notification, such as a permission prompt or input left idle | `message` |
| `turn-end` | `Stop` | right before Claude Code concludes its response | |

The edit tools are `Edit`, `Write` and `MultiEdit`; the read tool is `Read`. When a post-edit command or a validator
fails, `the-cook hook` exits 2, and Claude Code shows the hook's stderr to the model at once, so its next step can fix
the problem. The `attention` and `turn_end` actions do nothing in an attempt, whose worker runs with `COOK_ATTEMPT=1`.

### Permission decisions

- A matching `deny` wins over `ask`, and `ask` over `allow`, whichever settings file holds each rule.
- A command no rule matches falls to the session's mode: in manual mode Claude Code prompts; in auto mode its
  classifier decides from the conversation, without a prompt.
- A matching `ask` prompts even in auto mode, and no `allow` from any settings scope overrides it. In an attempt,
  `--permission-prompts none` denies it instead, and the worker reports the task as blocked.
- A matching `allow` skips the classifier: the command runs without judgement.

So production, cloud and repository-settings writes stay unlisted for Claude Code: a person in the session decides,
the classifier judges them in a session left alone, and attempts are denied them through `unattended.deny`
([placing an entry](https://github.com/lmarqs/the-cook/blob/main/docs/features/safety.md#placing-a-policy-entry)).

### Adoption

[`the-cook scaffold --adopt`](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#adopt) reads
Claude Code's rules from the files `native_rules` names for each
[import scope](https://github.com/lmarqs/the-cook/blob/main/docs/features/scaffold.md#import-scopes):

| Scope | File | What becomes of it |
| --- | --- | --- |
| `project` | `.claude/settings.json` | its `permissions.allow`, `ask` and `deny`, committed as the repository's own policy |
| `local` | `.claude/settings.local.json` | the same lists, each entry marked for review in the to-do, because one person's rules become shared |
| `user` | `~/.claude/settings.json` | the same, marked for review |

A rule shaped `Bash(<words> *)` comes back as a prefix list under the same decision, such as `[npm, run, test]`,
rendered for every worker. Every other rule, a `Bash(...)` with a `*` inside, `WebFetch(domain:...)`, `Read(...)`,
`Edit(...)`, a tool name on its own or an `mcp__` name with globs, becomes a native rule string, rendered verbatim for
Claude Code only. Each entry carries a provenance comment, such as `# imported from .claude/settings.json (project)`. A
hook becomes a declaration in `hooks.yaml` when it has a shape `hooks.yaml` models, otherwise a `raw` entry under its
event, which keeps running as written. Adoption leaves in place, and reports as notes, the keys that change how
permissions behave without being rules: `permissions.defaultMode`, `permissions.additionalDirectories`,
`permissions.disableBypassPermissionsMode`, `autoMode` and `sandbox`. It drops a rule that would weaken a floor rule,
such as an allow for a force push to the default branch, and names it.

## Floor

```yaml
floor:
  bypass:
    - --dangerously-skip-permissions
    - --allow-dangerously-skip-permissions
    - bypassPermissions
    - --permission-mode=bypassPermissions
```

| Argument | What it does in Claude Code |
| --- | --- |
| `--dangerously-skip-permissions` | bypasses every permission check |
| `--allow-dangerously-skip-permissions` | makes bypassing available as an option the session can switch to |
| `bypassPermissions` | the permission mode with no prompts, as the value in `--permission-mode bypassPermissions` |
| `--permission-mode=bypassPermissions` | the same mode, in one argument |

`the-cook attempt`, `attempt` actions and recipe `attempt` steps refuse each of them wherever it appears among the
arguments they receive, with exit 1 naming the argument, so a loop runner configured to pass one fails at its first task
([the attempt contract](https://github.com/lmarqs/the-cook/blob/main/docs/features/attempts.md#the-attempt-contract)).
The floor matches whole arguments: it refuses the spellings it lists, and the arguments under
[Arguments that change which settings load](#arguments-that-change-which-settings-load-are-not-refused) pass. The
ingredient denies no command of its own; the floor deny rules of the other pinned ingredients render into
`permissions.deny`.

## Checks

| Check | Value |
| --- | --- |
| `version` | `claude --version`, which prints `2.1.296 (Claude Code)`; supported: `>=2.1.292 <2.2` |
| `detect` | `.claude/`, `CLAUDE.md` |

`./cook doctor` reports a Claude Code outside the range: install a version inside it, or pin a tag of this cookbook
whose ingredient supports yours. `./cook ingredient list --cookbook github:lmarqs/the-cookbook@v0.1.0` shows the
ingredient as detected in a repository with either path, so a repository that uses Claude Code without pinning it, and
so without its floor, finds out.

## Pitfalls

### Claude Code does not read `AGENTS.md`

- **Symptom.** Sessions ignore the conventions in `AGENTS.md`, its managed blocks included, and never read the
  repository's memory.
- **Why.** Claude Code reads `CLAUDE.md`, and loads `AGENTS.md` by itself only in a project that has no `CLAUDE.md`.
- **What the ingredient does.** Renders the `import` block with `@AGENTS.md` into `CLAUDE.md`; `./cook render` puts it
  back after an edit removed it.

### An `ask` entry stops a session left alone

- **Symptom.** A Claude Code session in auto mode, left alone, stops and waits; or an attempt reports a command as
  denied.
- **Why.** A matching `ask` prompts even in auto mode, and no `allow` overrides it. An attempt has nobody to answer, and
  `--permission-prompts none` denies it.
- **What the ingredient does.** Renders `ask` as written. Keep `ask` for what needs a person every time, such as the
  seeded `[the-cook, lifecycle, run, deliver]`: each one costs a prompt in every session left alone.

### An `allow` entry skips the classifier

- **Symptom.** A session in auto mode runs a command you expected it to judge, such as a push to the default branch
  under `allow: [[git]]`.
- **Why.** An `allow` decides before the classifier is consulted; the classifier judges only what no rule matches.
- **What the ingredient does.** Renders the entries, nothing more. The [git](git.md) ingredient's floor still denies a
  force push to the default branch, because a deny wins. Write each `allow` as narrow as its reason, and rely on branch
  protection for what reaches the default branch.

### A flag before the subcommand reaches the classifier

- **Symptom.** The policy denies `[kubectl, patch]`, yet `kubectl -n prod patch deployment api` is left to the
  classifier.
- **Why.** `Bash(kubectl patch *)` matches the commands that start with `kubectl patch`; a flag in between makes another
  command.
- **What the ingredient does.** Renders prefixes exactly. Where Claude Code must stop the flag-first spelling too, add
  a bare string such as `"kubectl * patch *"`, which only Claude Code enforces.

### Settings outside the policy widen or narrow permissions

- **Symptom.** A command runs, or prompts, against what the policy says.
- **Why.** Claude Code also reads `.claude/settings.local.json`, the user's `~/.claude/settings.json` and managed
  settings, and combines their rules with the project's.
- **What the ingredient does.** Never writes those files. `scaffold --adopt --import local,user` reads them and marks
  each imported rule for review, and adoption reports the keys under [Adoption](#adoption) as notes. Before a session
  is left alone, check that none of them widens or narrows what it needs.

### An attempt follows the person's own plugins and instructions

- **Symptom.** The same task runs differently in attempts started on two machines.
- **Why.** A headless Claude Code session loads the same context as an interactive one, including the person's own
  settings, plugins, hooks and instructions under `~/.claude`, unless it is started with `--bare`
  ([headless runs](https://code.claude.com/docs/en/headless.md), checked against 2.1.296).
- **What the ingredient does.** Nothing: the person's own configuration is theirs, and it applies in the attempts
  `the-cook` starts for them
  ([ADR 0028](https://github.com/lmarqs/the-cook/blob/main/docs/adrs/0028-place-of-the-tool.md)). A worker declared
  with a jail starts without the person's home, so only what the jail's options let through reaches it.

### Project settings apply only in a trusted folder

- **Symptom.** In a session with a person, every command prompts, the project's hooks do not run, or Claude Code does
  not list the shipped skill.
- **Why.** An interactive Claude Code session applies project settings only in a folder the person has trusted.
  `claude --print`, which every attempt runs, skips the trust dialog.
- **What the ingredient does.** Nothing about trust: `the-cook` never changes it. Trust each checkout where you start
  sessions, the primary checkout and a workspace that asks.

### An attempt ignores a settings file that fails validation

- **Symptom.** An attempt runs a command that the rendered `permissions.deny` forbids, such as a floor rule, or gets no
  post-edit feedback.
- **Why.** In `--print` mode, Claude Code silently ignores a settings file that fails validation, such as a
  `.claude/settings.json` whose hand edit it rejects.
- **What the ingredient does.** Passes `unattended.deny` as `--disallowedTools`, which holds without the file; the
  floor and shared rules live only in the file. `./cook render --check` reports a hand edit to the owned keys, and an
  interactive `claude` started in the checkout shows a validation error.

### A loop runner's bypass flag stops the run at its first task

- **Symptom.** The first task fails, and `the-cook` exits 1 naming `--dangerously-skip-permissions`.
- **Why.** ralph-tui's `claude` plugin adds `--dangerously-skip-permissions` while its `skipPermissions` option is on,
  which is its default, and a command that forwards the plugin's arguments passes it on.
- **What the ingredient does.** Refuses it, as a floor bypass argument. Turn the option off, or drop the plugin's
  arguments, as the command of the [agentic-coding](../stations/agentic-coding.md) station does
  ([ralph-tui](ralph-tui.md#agent-plugins-turn-permission-checks-off-by-default)).

### Arguments that change which settings load are not refused

- **Symptom.** An attempt runs without the project's hooks, or under rules other than the rendered ones.
- **Why.** `--setting-sources` chooses which of the user, project and local settings load, `--settings` loads another
  file or JSON on top of them, `--restricted` ignores the user, project and local settings files, and `--bare` skips
  hooks and `CLAUDE.md`. None of them turns the permission checks off, so none is a floor argument.
- **What the ingredient does.** Passes `unattended.deny` at launch, which holds whatever settings load. Keep these
  arguments out of every configuration that starts `the-cook attempt` or `the-cook make`.

### `attention` fires for notifications that are not about a person

- **Symptom.** An `attention` action runs after a login or a usage notice, not only when Claude Code waits.
- **Why.** Claude Code's `Notification` event carries every notification type: `permission_prompt` and `idle_prompt`,
  and others such as `auth_success` and notices about usage limits.
- **What the ingredient does.** Maps `attention` to `Notification`, with no matcher. `COOK_MESSAGE` carries the
  notification's text, for an action that needs to tell them apart.

### A rule written into `.claude/settings.json` disappears

- **Symptom.** A permission rule added by hand to `.claude/settings.json` is gone after a render, or
  `./cook render --check` reports the file as edited by hand.
- **Why.** `permissions.allow`, `ask` and `deny` are owned keys.
- **What the ingredient does.** Writes them on every render. Put the rule in `.cook/policy.yaml`; a personal rule
  belongs in `.claude/settings.local.json`, which `the-cook` never writes.

### A worker can change the settings the ingredient does not own

- **Symptom.** After a sequential run, `.claude/settings.json` holds a hook entry, an `env` value or a
  `permissions.defaultMode` that no person wrote.
- **Why.** The tamper check restores and compares only the owned keys and entries. A verified attempt commits the rest
  of the worker's changes, and the next task's attempt starts from that commit.
- **What the ingredient does.** Owns the permission lists and its own hook entries only, so the hooks and settings a
  person or a plugin adds keep working. Read the changes to `.claude/settings.json` in each task commit before you
  deliver it.

### Auto mode is not available to every session

- **Symptom.** Claude Code reports that auto mode is unavailable: for the plan, for the model, or while fast mode is
  on.
- **Why.** Claude Code does not offer auto mode to every account and model.
- **What the ingredient does.** Pins auto mode and no other. Check that `claude --permission-mode auto` works for the
  account and the model the attempts use, and keep fast mode off for them.

### A prompt passed as an argument can be read as a rule

- **Symptom.** An attempt started with its prompt as an argument fails for lack of input, or denies a tool named after
  a word of the prompt.
- **Why.** `--disallowedTools` takes a list of rules, so an argument without a leading `-` after it can be read as one
  more. The list is split at each space or comma outside parentheses.
- **What the ingredient does.** Passes the prompt of `attempt` steps and actions on stdin, and renders every rule with
  its spaces inside `Bash(...)`. Have a loop runner pass its prompt on stdin too.

### Notebook edits run no post-edit command

- **Symptom.** Claude Code edits a `.ipynb` file, and no `post_edit` command runs.
- **Why.** Claude Code edits notebooks with its `NotebookEdit` tool, whose input names the file as `notebook_path`.
- **What the ingredient does.** Maps `Edit`, `Write` and `MultiEdit` as the edit tools, with the file at
  `tool_input.file_path`. The Verify stage checks the notebook with the rest of the change.

### Claude Code's own worktrees are not managed

- **Symptom.** Worktrees under `.claude/worktrees/` are not listed, cleaned or bootstrapped by `the-cook workspace`.
- **Why.** `claude --worktree` creates them, and they belong to Claude Code.
- **What the ingredient does.** Nothing: `the-cook` never scans or changes `.claude/worktrees/`. Use
  `./cook workspace create` for work that needs a task id, a receipt base or the repository's `bootstrap`.

### Rules written with `:*` stay Claude-only after adoption

- **Symptom.** After `scaffold --adopt`, a rule such as `Bash(npm run test:*)` is a native rule string, marked as not
  enforced in codex.
- **Why.** Claude Code reads both `Bash(<words> *)` and `Bash(<words>:*)` as a command prefix, and the ingredient reads
  only the first shape back as a prefix list.
- **What the ingredient does.** Imports the rule as it is, so Claude Code keeps enforcing it. Rewrite it as a prefix
  list, `[npm, run, test]`, so every worker enforces it.

### Claude Code prompts where Codex runs

- **Symptom.** Claude Code prompts for a command a Codex session runs on its own.
- **Why.** Claude Code's permissions decide each tool call, while Codex runs commands inside its sandbox and its rules
  decide what leaves it.
- **What the ingredient does.** Renders the same policy for both. Rely on `deny` and the classifier in Claude Code, and
  on the sandbox in Codex ([codex](codex.md#pitfalls)).

## Examples

### Pin it and declare a worker

```sh
./cook ingredient pin claude --cookbook github:lmarqs/the-cookbook@v0.1.0
```

Pinning declares no worker. Declare it in the profile, then render:

```yaml
# .cook/config.yaml
ingredients:
  - name: claude
    source: github:lmarqs/the-cookbook
    version: v0.1.0
    sha256: 0000000000000000000000000000000000000000000000000000000000000000
workers:
  claude: { role: interactive }
```

```sh
./cook render
git diff .claude/settings.json CLAUDE.md AGENTS.md
```

The [agentic-coding](../stations/agentic-coding.md) station pins the ingredient and declares the same worker, as
`claude: { ingredient: claude, role: interactive }`, beside an unattended [`codex`](codex.md) worker. Deleting the
worker from `workers` and running `./cook render` removes its hook entries, permission lists, link and block.

### A repository where only Claude Code works

A Terraform repository whose people use Claude Code and run no other worker:

```yaml
workers: { claude: { role: interactive } }
```

Its attempts, such as an `attempt` action after a merge, still receive `unattended.deny` at launch.

### A read-only MCP tool without a prompt

```yaml
# .cook/policy.yaml
shared:
  allow:
    # Read-only GitHub tools; they change nothing.
    - "mcp__github__get_*"
```

The rule renders verbatim into `permissions.allow`. Codex has no equivalent, and the `policy` block of `AGENTS.md`
marks it as not enforced in codex.

### Adopting hand-written settings

A Go repository whose `.claude/settings.json` allows `Bash(go test *)` and `WebFetch(domain:pkg.go.dev)` and denies
`Bash(goreleaser release *)`:

```sh
the-cook scaffold --cookbook github:lmarqs/the-cookbook@v0.1.0 --station agentic-coding --station go --adopt --dry-run
```

The generated `.cook/policy.yaml` holds `[go, test]` and `"WebFetch(domain:pkg.go.dev)"` under `shared.allow` and
`[goreleaser, release]` under `shared.deny`, each with its provenance comment, beside the station's `unattended.deny`;
a `defaultMode` in the file is reported as a note. After the same command with `--yes`, a session started with
`claude "finish adopting the-cook"` resolves the to-do with the shipped skill, and `./cook doctor` shows what is left.

### A coordinating session

```sh
claude --permission-mode auto
```

A session started in the primary checkout this way can create a run's workspace, start the loop runner headless, watch
it, and fire `completed` or `blocked` with scope `run` when it ends. Its own policy must allow what the agreed delivery
needs; the seeded `ask` on `[the-cook, lifecycle, run, deliver]` stops it for a person before delivery. The
[coordinated run](../use-cases/coordinated-run.md) use case walks through one.
