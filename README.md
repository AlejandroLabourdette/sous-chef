# sous-chef

**Talk to one agent. Cook with a brigade.**

sous-chef turns a Claude Code session into an orchestrator. You hand it a task, it opens an
isolated **station** for it, and it comes straight back to you ready for the next one. Stations
run in parallel and never collide, because each one is a separate git worktree with its own live
Claude Code session.

```
station = git worktree + herdr workspace + a chef-de-partie session + branch <type>/<slug>
```

The chef-de-partie in each station is a **real interactive session**, not a background subagent.
You can switch into its tab whenever you want, answer its questions, look at its work, and steer
it, then switch back. That is the point: parallel work you can still talk to.

## The five verbs

| Verb | What it does |
| --- | --- |
| `/fire <slug> <task>` | Open a station and hand it the ticket |
| `/brigade` | Live status of every station |
| `/pass <slug>` | Independent background review, delivered into that station's tab |
| `/plate <slug>` | Push the branch and open the pull request |
| `/86 <slug>` | Tear the station down |

You never have to type them. "open a station for the auth refactor", "how is the brigade doing",
"review the dark mode one", "ship it" all land on the same verbs.

## A service, end to end

```
you  ▸ /sous-chef
     ▸ /fire dark-mode add a dark theme toggle to the settings page

sous-chef ▸ station dark-mode is open on feat/dark-mode, workspace w4.
            It is planning and will need your approval in its own tab.

you  ▸ /fire flaky-login fix the flaky login test

sous-chef ▸ station flaky-login is open on fix/flaky-login, workspace w5.

     (two stations now cooking in parallel; you switch to w4, approve the plan,
      come back, switch to w5, answer a question, come back)

you  ▸ dark-mode looks good, review it

sous-chef ▸ reviewer running in the background. I am free in the meantime.
            ... review 1: 4 correctness findings, 3 quality ones. It is in
            dark-mode's tab, workspace w4; triage it there with the chef.

     (in the dark-mode tab you pick which findings to implement)

you  ▸ ship dark-mode

sous-chef ▸ this will push feat/dark-mode (7 commits) and open a PR against main. Confirm?
```

Nothing about that flow blocks. While a review runs you can fire another ticket, pass another
station, or ask for the brigade status.

## Requirements

- **[herdr](https://herdr.dev)** 0.8.x. sous-chef is built on it and will refuse to run outside
  it. Worktrees, station sessions and lifecycle state all come from herdr.
- **Claude Code** 2.1.x
- **git** and **gh**, with `gh auth login` done (needed by `/plate`)
- **jq**

## Install

```bash
git clone https://github.com/<you>/sous-chef
claude plugin marketplace add ./sous-chef
claude plugin install sous-chef
```

Install it, do not merely load it with `--plugin-dir`. Station sessions are separate `claude`
processes that resolve plugins on their own, so a `--plugin-dir` load would give the orchestrator
the skills and leave the stations without them.

Then, in any repository:

```bash
claude
> /sous-chef
```

### Optional: desktop notifications

Stations announce themselves through `herdr notification show`. herdr ships with toasts off, so
those calls are silently no-ops until you enable delivery in `~/.config/herdr/config.toml`:

```toml
[ui.toast]
delivery = "herdr"   # or "system" for OS notifications, "terminal" to ask your terminal
```

Without it everything still works; you just read station state from herdr's sidebar instead of
getting pinged.

## What it does not do

- **It does not merge.** `/plate` opens the pull request and stops. Merging is yours, with CI in
  front of you.
- **It does not push without asking.** `/plate` shows the exact branch, commits and PR body and
  waits for a yes.
- **It does not let a chef start writing on its own.** Every station starts in plan mode and has
  to get its plan approved in its own tab first. That plan is a list of atomic phases, and the
  chef implements one commit per phase, so the branch reads back as the plan you approved.
- **It does not name branches after itself.** A station branch is `<type>/<slug>` - `feat/`,
  `fix/`, `docs/` and the rest - so it reads like any other branch in your repository. sous-chef
  infers the type from the task and tells you which one it picked.
- **It does not touch your repository.** Tickets and review reports live in `~/.sous-chef/`,
  worktrees live in `~/.herdr/worktrees/`. No project ever needs a `.gitignore` entry for it.

## How it works

See [docs/architecture.md](docs/architecture.md).

## Prior art

[firstmate](https://github.com/kunchenguid/firstmate) solves the same problem with a nautical
crew analogy. It targets many harnesses and many multiplexers, so it hand-builds worktree
management, a bash supervision watcher, a session-backend abstraction and a state store.
sous-chef targets exactly one of each and gets all of that from herdr, which is why it is
instructions rather than code.
