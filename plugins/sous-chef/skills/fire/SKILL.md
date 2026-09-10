---
name: fire
description: Fire a ticket. Opens a new station for a task - a git worktree, a herdr workspace, and a live chef-de-partie Claude Code session started in plan mode - then hands it the brief. Use when the user wants to start a new parallel task, open a station, or delegate work to a chef-de-partie.
argument-hint: <slug> <task description>
---

# /fire

Open a station and hand it a ticket.

Arguments: `$ARGUMENTS`. The first token is the slug; everything after it is the task. If the
user gave a task but no slug, derive a short one and tell them what you picked.

## 1. Resolve and validate, in one call

The slug must match `^[a-z][a-z0-9_-]{0,31}$` - herdr's agent-name rule, not a style preference.
If it does not match, propose a corrected slug and ask before continuing.

Then pick the branch type from the task, using the table in the `sous-chef` skill. `$TYPE` and
`$SLUG` together are the branch.

Run the resolve preamble from the `sous-chef` skill and this tail as a **single** `Bash` call -
`/fire` needs the kitchen for the ticket path and the collision check needs `$REPO`:

```bash
TAKEN="$(git -C "$REPO" for-each-ref --format='%(refname:short)' \
  "refs/heads/$SLUG" "refs/heads/*/$SLUG" "refs/heads/**/$SLUG")"
AGENT="$(herdr agent get "$SLUG" >/dev/null 2>&1 && echo taken || echo free)"
jq -n --arg r "$REPO" --arg b "$BASE" --arg k "$KITCHEN/$SLUG" \
      --arg t "${TAKEN:-}" --arg a "$AGENT" \
  '{repo: $r, base: $b, station_dir: $k, agent_name: $a,
    branches_taken: ($t | split("\n") | map(select(length > 0)))}'
```

`$MATCHES` and `$ROWS` from the preamble are not interesting here - `/fire` is creating a station,
not resolving one - but running the whole preamble is what gives you `$REPO`, `$BASE` and
`$KITCHEN` in the same shell as the check that uses them.

Refuse to fire if `branches_taken` is non-empty or `agent_name` is `taken`. The branch check looks
for the slug under **any** prefix, not just the one you are about to use: a slug identifies a
station, so `fix/auth` blocks `feat/auth` - they would collide on the agent name and the station
directory regardless of type. It matches nested prefixes too, because `refs/heads/*/$SLUG` alone
does not: git's `*` does not cross a `/`, so a user's `wip/deep/auth` would slip past the check and
then make every later station lookup ambiguous, since those match on the suffix `/auth`.

If either hits, offer to focus the existing station or to pick another slug. Never reuse a slug
for a different task.

## 2. Write the ticket

Write `<station_dir>/ticket.md` before creating anything, so the brief survives a later failure.
The `## Station` block needs the worktree path, which does not exist until step 3, so write the
ticket now with that one line left as `<pending>` and fill it in after the worktree is created.
Do a little homework first and name the files and patterns the chef should start from. Do **not**
pre-solve the task or carve it into phases: both are the chef's job, done in plan mode with the
user watching.

```markdown
# Ticket: <slug>

## Task
<the user's request, in full, in their own terms>

## Context
<the handful of files, modules or patterns worth starting from, with paths>

## Acceptance criteria
- <what has to be true for this to be done>

## Out of scope
- <what this station must not touch, especially work owned by other stations>

## Station
- Branch: `<type>/<slug>`
- Base: `<base>`
- Worktree: `<pending until step 3>`
- Station directory: `<kitchen>/<slug>/`

## Protocol
Invoke the `chef-de-partie` skill and follow it for the whole life of this station. Plan in
atomic phases and implement one commit per phase.
```

**State the branch in the ticket.** It is how the chef learns its own branch without
reconstructing it. herdr names the worktree directory after the branch with `/` replaced by `-`
(`~/.herdr/worktrees/<repo-name>/<type>-<slug>`), but read it from the step 3 response rather than
assuming, and edit it into the ticket there.

## 3. Create the worktree

Substitute the literal `repo` and `base` from step 1 - this is a new call and nothing is still set:

```bash
herdr worktree create --cwd <repo> --branch <type>/<slug> --base <base> --label <slug> --no-focus \
  | jq '{pane: .result.root_pane.pane_id,
         workspace: .result.workspace.workspace_id,
         worktree: .result.worktree.path}'
```

`--no-focus` is not optional. The user stays with you and chooses when to visit a station.

## 4. Start the chef-de-partie

```bash
herdr agent start <slug> --kind claude --pane <pane> -- \
  --permission-mode plan -n <slug> --add-dir "$HOME/.sous-chef"
```

- `--permission-mode plan` is the core of the contract: the chef cannot write until the user
  approves its plan in the station's own tab.
- `-n <slug>` names the session for the tab, the title and `/resume`.
- `--add-dir "$HOME/.sous-chef"` lets the chef read its ticket and reviews, which live outside
  the worktree. The Preflight already created that directory.
- No `--timeout`: herdr's 30s default is enough, and this call is the one point where sous-chef
  waits at all. If the session is not ready by then you get `agent_not_ready`, which is handled
  just below - a longer wait buys nothing the failure path does not already cover.

### If it returns `agent_not_ready`

The session started but is sitting at a dialog. Identify it, never answer it blindly:

```bash
herdr agent read <slug> --source detection --lines 40
```

The common case is Claude Code's workspace-trust prompt, which appears at most once per
repository. Tell the user that, and that one confirmation in the station's tab unblocks it. Offer
`herdr workspace focus <workspace>`. Anything else at that prompt: report what you saw verbatim and
ask. Never answer a security dialog on the user's behalf.

## 5. Hand over the ticket

```bash
herdr agent prompt <slug> "You are the chef-de-partie for station <slug>. Read <station_dir>/ticket.md, then invoke the chef-de-partie skill and follow it."
```

Do not pass `--wait`. The chef will be working for a while and you must stay available.

## 6. Report back

Tell the user the slug, the branch, the workspace id to switch to, and that the station is
planning and will need their approval in its own tab. Then stop. Do not poll the station.

**Name the type you inferred.** You chose it on their behalf, so this report is the only place
they see the decision, and if it is wrong the fix is cheap: `/86` and fire again, before the chef
has built anything.

```bash
herdr notification show <slug> --body "station open, planning" --sound done
```
