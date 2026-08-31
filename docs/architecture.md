# Architecture

## The shape of the thing

sous-chef is **instructions, not code**. It ships no scripts, no daemon and no state store. It is
a set of skills that teach a Claude Code session to drive three tools it already has:

```
sous-chef (skills)  ──▶  herdr   worktrees, workspaces, station sessions, lifecycle state
                    ──▶  git     branches, diffs, the shared object store
                    ──▶  gh      pull requests
```

That is a deliberate choice, and it is the main difference from
[firstmate](https://github.com/kunchenguid/firstmate), which solves the same problem. firstmate
supports six agent harnesses and five terminal multiplexers, so it has to build the common layer
itself. sous-chef supports one of each, and that layer already exists:

| firstmate builds | sous-chef uses |
| --- | --- |
| `treehouse` for worktrees | `herdr worktree create --branch X --base Y` |
| a tmux window per crewmate | `herdr agent start <name> --kind claude --pane <id>` |
| a bash watcher polling for state | `herdr agent list` -> `working` / `blocked` / `idle` / `done` |
| a durable state store | `herdr worktree list` + `herdr agent list` + git refs |
| a backend abstraction over 5 multiplexers | nothing; herdr only |
| notification plumbing | `herdr notification show` |
| harness session correlation | `herdr agent list` returns the Claude Code session id |

Every one of those rows is a mechanism that does not exist here, and therefore cannot drift,
break on upgrade, or lie about the state of the world.

## A station

```
station
├── git worktree      ~/.herdr/worktrees/<repo>/sous-chef-<slug>   (created by herdr)
├── herdr workspace   labelled <slug>, its own tab in the sidebar
├── chef-de-partie    a real `claude` process in that workspace's root pane
├── branch            sous-chef/<slug>
└── station directory ~/.sous-chef/<repo>-<hash>/<slug>/  (ticket.md, plan.md, review-N.md)
```

The chef-de-partie is an **interactive session, not a subagent**. That is the requirement the
whole design falls out of: the user has to be able to switch into a station, answer its
questions, and review its work. Subagents cannot be talked to; a `claude` process in a herdr pane
can.

## Two invariants

### 1. Live state is composed, never stored

There is no station registry. `/brigade` derives everything from
`herdr worktree list --cwd <repo>`, `herdr agent list` and `git`, joining them on the worktree
path. A stored index would eventually disagree with reality about which stations exist, and a
status report that lies is worse than no status report.

What *is* stored is durable intent only: the ticket you wrote when firing, the phase plan the
user approved, and the review reports. Those cannot be recomputed, so they live on disk under
`~/.sous-chef/<repo>-<hash>/<slug>/`. Keying by a hash of the repository path means two checkouts
of the same project never collide.

`plan.md` is intent, not state, which is why it does not break the invariant. It records the
phases the user agreed to, and it is never updated to say how far along the station is. Progress
is read from `git log`, where one commit per phase makes it unambiguous.

Nothing is ever written inside the user's repository.

### 2. The orchestrator never blocks

The user must always be able to fire another ticket. So:

- Reviews run as **background subagents**. `/pass` dispatches and returns immediately.
- Stations are never waited on. They report back themselves, with one
  `herdr agent prompt sous-chef` ping per event.
- The orchestrator renames its own pane to `sous-chef` at startup, which is what makes that
  reverse channel addressable by name.

## Why plans are phased

A station's plan is a sequence of atomic phases, and each approved phase becomes exactly one
commit. The rule lives in the `chef-de-partie` contract, and it exists to serve the two verbs
downstream of it.

`/pass` reviews a branch, and a branch whose history is one large commit gives a reviewer no
seams to reason about: every change arrives at once, with no statement of which change was meant
to do what. Phases give the reviewer the author's own decomposition to check the diff against,
which is why the reviewer brief reads `plan.md` and treats a mismatch as a finding.

`/plate` then gets a pull request body it does not have to invent. The phases already describe the
change at the level of behaviour, in order, and they map one to one onto the commit list.

The cost is paid entirely at plan time, in a tab where the user is already reading the plan, and
it is the cheapest possible moment to notice that a change is really three changes.

## The naming convention is the API

One slug drives every identifier, which is what makes state composable without a registry:

| Thing | Value |
| --- | --- |
| slug | `^[a-z][a-z0-9_-]{0,31}$`, herdr's agent-name rule |
| branch | `sous-chef/<slug>` |
| herdr agent name | `<slug>` |
| herdr workspace label | `<slug>` |
| worktree path | `~/.herdr/worktrees/<repo>/sous-chef-<slug>` |
| station directory | `~/.sous-chef/<repo>-<hash>/<slug>/` |

## Why the reviewer works from the main repository

`/pass` reviews `sous-chef/<slug>` **without entering the worktree**. A linked worktree shares the
main repository's object store, so the branch is an ordinary ref:

```bash
git -C <REPO> diff <BASE>...sous-chef/<slug>
git -C <REPO> show sous-chef/<slug>:<path>
```

This avoids cross-directory permission friction entirely, and it means the reviewer never has to
be granted access to a directory outside the project it was launched in.

The review itself delegates to the existing `code-review` skill rather than reimplementing
severity ranking and false-positive filtering.

## Verified herdr behaviour

Confirmed against herdr 0.8.2 and Claude Code 2.1.236. These are the details the skills depend
on.

**`herdr worktree create` response shape:**

```
.result.root_pane.pane_id        the shell pane to start the chef in
.result.workspace.workspace_id   for focusing and for removal
.result.worktree.path            the checkout path
```

**`herdr agent start` gives back the Claude session id** at
`.result.agent.agent_session.value`, which correlates a station with its Claude Code transcript.

**Panes cannot be read for content.** Claude Code runs on the terminal's alternate screen, so
`herdr agent read` cannot recover a station's replies; rows that leave the alternate screen never
enter herdr's scrollback. This is why everything that travels between stations and the
orchestrator is a file under `~/.sous-chef` or a one-line prompt, never scraped output.
`agent read --source detection` is still useful for one thing: identifying a blocking dialog.

**The workspace-trust dialog is per repository, not per station.** A station started in a
repository Claude Code has not trusted yet returns `agent_not_ready` and sits at the trust
prompt. Accepting it records trust against the **main repository root**, so every later station
in that project starts clean. In practice the orchestrator is already running in that repository,
so it is already trusted and stations start with `interactive_ready: true`.

There is no supported way to pre-accept it: no `CLAUDE_CODE_*` environment variable exists for
it, and `hasTrustDialogAccepted` lives in `~/.claude.json`, a file Claude Code owns and rewrites,
where a concurrent write would race. So sous-chef identifies the dialog and asks the user for one
keystroke instead of answering a security prompt on their behalf.

**Notifications are off by default.** `herdr notification show` returns
`{"reason":"disabled","shown":false}` unless `[ui.toast] delivery` is set in
`~/.config/herdr/config.toml`. The calls are harmless no-ops when disabled.

**Stations must be started with `--permission-mode plan`.** Verified: the station's footer reads
`plan mode on` and it cannot write until the user approves its plan in that tab.
