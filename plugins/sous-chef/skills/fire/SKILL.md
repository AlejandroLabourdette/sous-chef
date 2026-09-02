---
name: fire
description: Fire a ticket. Opens a new station for a task - a git worktree, a herdr workspace, and a live chef-de-partie Claude Code session started in plan mode - then hands it the brief. Use when the user wants to start a new parallel task, open a station, or delegate work to a chef-de-partie.
argument-hint: <slug> <task description>
---

# /fire

Open a station and hand it a ticket. In the kitchen, firing an order is starting to cook it.

Arguments: `$ARGUMENTS`. The first token is the slug; everything after it is the task. If the
user gave a task but no slug, derive a short one from the task and tell them what you picked.

## 1. Resolve and validate

Resolve `REPO`, `BASE` and `KITCHEN` as described in the `sous-chef` skill.

The slug must match `^[a-z][a-z0-9_-]{0,31}$`. This is herdr's agent-name rule, not a style
preference: a slug outside it cannot be used as an agent name. If it does not match, propose a
corrected slug and ask before continuing.

Then pick the branch type from the task, using the table in the `sous-chef` skill. `$TYPE` and
`$SLUG` together are the branch.

Refuse to fire if the station already exists:

```bash
git -C "$REPO" for-each-ref --format='%(refname:short)' "refs/heads/$SLUG" "refs/heads/*/$SLUG"
herdr agent get "$SLUG" >/dev/null 2>&1 && echo "agent name taken"
```

The branch check looks for the slug under **any** prefix, not just the one you are about to use.
A slug identifies a station, so `fix/auth` blocks `feat/auth`: they would collide on the agent
name and the station directory regardless of type.

If either hits, the station is already open. Offer to focus it or to pick another slug. Never
reuse a slug for a different task.

## 2. Write the ticket

Write `$KITCHEN/$SLUG/ticket.md` before creating anything, so the brief exists even if a later
step fails.

Do a little homework first: name the files and existing patterns the chef should start from.
Do **not** pre-solve the task, and do not carve it into phases. Designing the change and cutting
it into phases are both the chef's job, and it does them in plan mode with the user watching.

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
- Worktree: `<path>`
- Station directory: `<kitchen>/<slug>/`

## Protocol
Invoke the `chef-de-partie` skill and follow it for the whole life of this station. Plan in
atomic phases and implement one commit per phase.
```

Fill the worktree path in after step 3, or write the ticket in two passes. herdr names the
directory after the branch, with `/` replaced by `-`
(`~/.herdr/worktrees/<repo-name>/<type>-<slug>`), but read it from the response rather than
assuming.

State the branch in the ticket. It is how the chef learns its own branch without reconstructing
it.

## 3. Create the worktree

```bash
OUT="$(herdr worktree create --cwd "$REPO" --branch "$TYPE/$SLUG" --base "$BASE" --label "$SLUG" --no-focus)"
PANE="$(printf '%s' "$OUT" | jq -r .result.root_pane.pane_id)"
WS="$(printf '%s' "$OUT" | jq -r .result.workspace.workspace_id)"
WT="$(printf '%s' "$OUT" | jq -r .result.worktree.path)"
```

`--no-focus` is not optional. The user stays with you; they choose when to visit a station.

## 4. Start the chef-de-partie

```bash
mkdir -p "$HOME/.sous-chef"
herdr agent start "$SLUG" --kind claude --pane "$PANE" --timeout 60000 -- \
  --permission-mode plan -n "$SLUG" --add-dir "$HOME/.sous-chef"
```

Why each flag:

- `--permission-mode plan` is the core of the contract. The chef explores and proposes a plan,
  and cannot write anything until the user approves it in the station's own tab.
- `-n "$SLUG"` names the session so it is identifiable in the tab, the title and `/resume`.
- `--add-dir "$HOME/.sous-chef"` lets the chef read its ticket and its review reports, which
  live outside the worktree.

A successful start returns `.result.agent`, including `.agent_session.value`, the Claude Code
session id for that station. Mention it only if the user asks; the slug is the handle.

### If it returns `agent_not_ready`

The session started but is sitting at a dialog. Identify it, never answer it blindly:

```bash
herdr agent read "$SLUG" --source detection --lines 40
```

The common case is Claude Code's workspace-trust prompt ("Accessing workspace... Is this a
project you created or one you trust?"). Trust is recorded per repository, so this appears at
most once per project and not once per station. Tell the user exactly that, and that one
confirmation in the station's tab unblocks it. Offer `herdr workspace focus "$WS"`. Do not send
keys to answer a security dialog on the user's behalf.

Anything else at that prompt: report what you saw verbatim and ask.

## 5. Hand over the ticket

```bash
herdr agent prompt "$SLUG" "You are the chef-de-partie for station $SLUG. Read $KITCHEN/$SLUG/ticket.md, then invoke the chef-de-partie skill and follow it."
```

Do not pass `--wait`. The chef will be working for a while and you must stay available.

## 6. Report back

Tell the user, in a couple of lines: the slug, the branch, the workspace id to switch to, and
that the station is planning and will need their approval in its own tab. Then stop. Do not
poll the station.

Name the type you inferred. You chose it on their behalf, so the report is the only place they
see the decision, and if it is wrong the fix is cheap: `/86` and fire again, before the chef has
built anything.

```bash
herdr notification show "$SLUG" --body "station open, planning" --sound done
```
