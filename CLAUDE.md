# sous-chef

A Claude Code plugin that orchestrates parallel work: one **station** per task, where a station
is a git worktree plus its own live Claude Code session, managed through **herdr**.

## Where things are

- `plugins/sous-chef/skills/sous-chef/` - the orchestrator protocol. Start here.
- `plugins/sous-chef/skills/chef-de-partie/` - the station-side contract.
- `plugins/sous-chef/skills/{fire,brigade,pass,plate,86}/` - one skill per verb.
- `docs/architecture.md` - why it is built this way, and the verified herdr behaviours it relies
  on.

## The bar for changes here

**This plugin is instructions, not code.** Before adding a script, a wrapper or a state file,
check whether herdr or git already provides it. The reason sous-chef is small is that
`herdr worktree`, `herdr agent` and git refs already cover worktrees, session lifecycle,
addressing and state. Every mechanism this repository owns is a mechanism that can drift from
reality.

Two invariants that carry most of the design:

- **Live state is never stored.** It is composed from `herdr worktree list`, `herdr agent list`
  and git on every read. Only durable intent (tickets, review reports) is written to disk, under
  `~/.sous-chef/`.
- **sous-chef never blocks.** Reviews are background subagents; stations report back on their
  own. Any change that makes the orchestrator wait on a station is a regression.

## Verifying a change

`claude plugin validate plugins/sous-chef` catches manifest and frontmatter errors, but it cannot
tell you whether the instructions work. Behaviour changes need a real run: a throwaway repo, a
sous-chef session inside herdr, and an actual `/fire` through `/plate`. `docs/architecture.md`
records the herdr response shapes and failure modes already confirmed that way.
