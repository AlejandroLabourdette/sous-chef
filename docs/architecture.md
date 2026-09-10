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
├── git worktree      ~/.herdr/worktrees/<repo>/<type>-<slug>      (created by herdr)
├── herdr workspace   labelled <slug>, its own tab in the sidebar
├── chef-de-partie    a real `claude` process in that workspace's root pane
├── branch            <type>/<slug>
└── station directory ~/.sous-chef/<repo>-<hash>/<slug>/  (ticket.md, plan.md, review-N.md,
                                                            pr-body.md)
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
user approved, the review reports, and the pull request body `/plate` shipped. Those cannot be recomputed, so they live on disk under
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
| branch | `<type>/<slug>`, type inferred from the ticket |
| herdr agent name | `<slug>` |
| herdr workspace label | `<slug>` |
| worktree path | `~/.herdr/worktrees/<repo>/<type>-<slug>` |
| station directory | `~/.sous-chef/<repo>-<hash>/<slug>/` |

The branch is the one row that is *not* derived from the slug, because its prefix says what kind
of work the station is doing and only the ticket knows that. It is looked up rather than rebuilt:
one row of `herdr worktree list --cwd <repo>`, selected on a branch ending in `/<slug>`, carries
the branch, the checkout path and the workspace id together. That is the same composition
`/brigade` already performs, which is why a semantic prefix cost no registry - the mapping was
never stored, only recomputed from a different column.

`/brigade` recognises its own worktrees the same way, by joining them against the station
directories that already exist under `~/.sous-chef`. Recognising them by branch prefix instead
would sweep in the user's own `feat/*` branches, now that stations follow the ordinary convention.

## Why each verb is one shell call

Every verb resolves the repository, the base, the kitchen and the station, and runs its own safety
checks, in a **single** `Bash` invocation that ends in one JSON object.

That is not an optimisation, though it is also that - it takes `/pass`, `/plate` and `/86` from four
calls to one, and `/brigade` from two-per-station to two. It is a correctness requirement. Shell
state does not survive between tool calls: a variable assigned in one call reads back empty in the
next. And the guards these verbs depend on fail *open* when that happens:

```bash
git -C "$WT" status --porcelain    # "must be empty"
```

With `$WT` empty this is `git -C ""`, which git resolves to the current directory - the
orchestrator's own checkout. It exits 0 and prints nothing, so the check reports a clean tree
without ever having looked at the station. That is the guard that stops `/pass` reviewing
uncommitted work and stops `/86` destroying it.

Keeping the whole preamble in one shell removes the failure mode rather than documenting around it.
The prose in each skill still says why each check exists; only the plumbing was collapsed.

The one field that has to be read before any other is `matches`. A slug does not name a branch, so
the branch is looked up by suffix, and a lookup that matched twice would otherwise put two
newline-separated branch names into `git push` and `git branch -d`.

## Comparing against a base that may have no remote

`BASE` is resolved from `origin/HEAD`, then `gh`, then the current branch. That last fallback yields
a plain local branch name with nothing remote behind it, so `origin/<base>` need not exist.

`git merge-base --is-ancestor origin/main <branch>` then exits **128**, and 128 is not 1: "I cannot
compare these" is a different answer from "the branch is behind". Reading 128 as "behind" sends a
station off to rebase onto a ref that does not exist.

So the preamble resolves a `BASEREF` - `origin/<base>` when that ref exists, `<base>` otherwise -
and every comparison uses it. `/plate` still distinguishes the three exit codes, because 128 can
also mean something else went wrong, and reports that it cannot tell rather than guessing.

## Why the reviewer works from the main repository

`/pass` reviews the station's branch **without entering the worktree**. A linked worktree shares
the main repository's object store, so the branch is an ordinary ref:

```bash
git -C <REPO> diff <BASE>...<BRANCH>
git -C <REPO> show <BRANCH>:<path>
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

**`herdr worktree list` reports `label` as the repository label.** Every row of
`herdr worktree list --cwd <repo>` carries the same `label` - the repository's - not the
per-station `--label` the worktree was created with. A lookup by label matches every station at
once, so stations are addressed by the branch suffix `/<slug>` instead.

**The worktree directory is named after the branch,** with `/` replaced by `-`. Verified across
three repositories, before and after the move to typed branches: branch `feat/demo` produces
`~/.herdr/worktrees/station/feat-demo`, and under the old convention branch
`sous-chef/philosophers-canon` in repo `sous-chef-test-enviroment` produced
`~/.herdr/worktrees/sous-chef-test-enviroment/sous-chef-philosophers-canon`. In both the repository
label appears nowhere in the leaf. So the branch type shows up in the path too, and the
path is always read from the `worktree create` response rather than predicted.

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

**`herdr worktree remove` needs a live workspace id.** A worktree whose workspace has been closed
reports no `open_workspace_id`, and `herdr worktree remove --workspace <anything> --force` answers
`{"error":{"code":"workspace_not_found"}}`. Verified by closing a workspace and retrying. That state
is reachable in normal use - it is what `/brigade` shows as `no session` - so `/86` falls back to
`git worktree remove` there. There is no workspace left to close, only a checkout to delete.

**`refs/heads/*/<slug>` does not match nested prefixes.** Verified: `git for-each-ref
'refs/heads/*/auth'` matches `feat/auth` but not `wip/deep/auth`, because git's `*` does not cross a
`/` in a ref pattern. `/fire` passes `refs/heads/**/<slug>` as well, otherwise a user branch two
levels deep slips past the collision check and then makes every station lookup ambiguous, since
those match on the suffix.
