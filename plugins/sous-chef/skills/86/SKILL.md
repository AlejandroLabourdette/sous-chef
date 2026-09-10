---
name: "86"
description: 86 a station. Tears down its worktree and workspace after checking nothing would be lost, and archives its ticket and reviews. Use when a station's pull request is merged, when the user abandons a task, or asks to close, remove or clean up a station.
argument-hint: <slug>
---

# /86

Remove a station: the worktree and its herdr workspace go away, the paperwork is archived. This
is the destructive verb. Check first, then act.

## 1. Refuse to destroy work silently

Resolve the station first, as described in the `sous-chef` skill. That one lookup gives you
`$BRANCH`, `$WT` and `$WS`, which is everything the teardown needs.

```bash
git -C "$WT" status --porcelain                                 # uncommitted work
git -C "$REPO" rev-list --count "origin/$BRANCH..$BRANCH" 2>/dev/null \
  || git -C "$REPO" rev-list --count "$BASE..$BRANCH"            # unpushed commits
```

If either shows something, stop and lay out precisely what would be lost: how many uncommitted
files, how many unpushed commits. Proceed only on an explicit yes to that specific loss, never on
a general "yes, clean it up" given before the user knew.

Also check the station is not still running:

```bash
herdr agent get "$SLUG" 2>/dev/null | jq -r .result.agent.agent_status
```

A `working` station is mid-task. Say so and ask before killing it.

## 2. Archive the paperwork

The ticket and its reviews are the record of why the change looks the way it does. Keep them:

```bash
mkdir -p "$KITCHEN/archive"
mv "$KITCHEN/$SLUG" "$KITCHEN/archive/$SLUG-$(date +%Y%m%d-%H%M%S)"
```

## 3. Remove the station

```bash
herdr worktree remove --workspace "$WS" --force
```

This closes the workspace, kills the pane and removes the git worktree in one step.

## 4. The branch is a separate decision

`herdr worktree remove` does not delete the branch, and that is the right default: it may be
under review or already pushed. Delete it only when the user asks, and check it is merged first:

```bash
git -C "$REPO" branch --merged "origin/$BASE" --list "$BRANCH"   # empty = not merged
git -C "$REPO" branch -d "$BRANCH"                               # -d, never -D
```

Use `-d`, never `-D`. If `-d` refuses, the branch has unmerged commits and the user needs to know
that rather than have it forced away.

## 5. Report

Say what was removed and what survived: the archived station directory, and the branch if it is
still there. If a pull request is open for that branch, remind the user it is still open.
