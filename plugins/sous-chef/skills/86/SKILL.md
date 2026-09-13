---
name: "86"
description: 86 a station. Tears down its worktree and workspace after checking nothing would be lost, and archives its ticket and reviews. Use when a station's pull request is merged, when the user abandons a task, or asks to close, remove or clean up a station.
argument-hint: <slug>
---

# /86

Remove a station: the worktree and its herdr workspace go away, the paperwork is archived. This
is the destructive verb. Check first, then act.

## 1. Refuse to destroy work silently

Run the resolve preamble from the `sous-chef` skill and this tail as a **single** `Bash` call.
This is the destructive verb, and every guard in it reads a path that must be live in the same
shell: `git -C "" status --porcelain` does not error, it reports on your own clean checkout, so a
split call would report "nothing to lose" about a station full of uncommitted work.

```bash
if [ "$MATCHES" = 1 ]; then
  BRANCH="$(printf '%s' "$ROWS" | jq -r '.[0].branch')"
  WT="$(printf '%s' "$ROWS" | jq -r '.[0].path')"
  WS="$(printf '%s' "$ROWS" | jq -r '.[0].open_workspace_id // "closed"')"
  DIRTY="$(git -C "$WT" status --porcelain | wc -l | tr -d ' ')"
  if git -C "$REPO" rev-parse --verify --quiet "origin/$BRANCH" >/dev/null 2>&1; then
    UNPUSHED="$(git -C "$REPO" rev-list --count "origin/$BRANCH..$BRANCH")"
  else
    UNPUSHED="$(git -C "$REPO" rev-list --count "$BASEREF..$BRANCH" 2>/dev/null)"
  fi
  MERGED="$(git -C "$REPO" branch --merged "$BASEREF" --list "$BRANCH" 2>/dev/null | wc -l | tr -d ' ')"
  CHEF="$(herdr agent list 2>/dev/null | jq -c --arg p "$WT" \
          '[(.result.agents // [])[] | select(.cwd == $p)] | .[0] // {}')"
  STATUS="$(printf '%s' "$CHEF" | jq -r '.agent_status // "no session"')"
fi
jq -n --argjson m "$MATCHES" --argjson rows "$ROWS" \
      --arg b "${BRANCH:-}" --arg w "${WT:-}" --arg ws "${WS:-}" --arg k "$KITCHEN/$SLUG" \
      --arg d "${DIRTY:-}" --arg u "${UNPUSHED:-}" --arg mg "${MERGED:-0}" --arg st "${STATUS:-}" \
  '{matches: $m, all_matches: [$rows[].branch], branch: $b, worktree: $w, workspace: $ws,
    station_dir: $k, uncommitted_files: $d, unpushed_commits: $u,
    merged: ($mg != "0"), agent_status: $st}'
```

Read it before removing anything:

- **`matches` is not 1** - stop. An ambiguous slug must never reach `herdr worktree remove` or
  `git branch -d`.
- **`uncommitted_files` or `unpushed_commits` is non-zero** - stop and lay out precisely what would
  be lost, by those numbers. Proceed only on an explicit yes to that specific loss, never on a
  general "yes, clean it up" given before the user knew.
- **`agent_status` is `working`** - the station is mid-task. Say so and ask before killing it. It is
  `no session` when no chef is running in the worktree, which is normal after a pane was closed.

`unpushed_commits` counts against `origin/<branch>` when the branch was pushed, and against the
base otherwise - where every commit on the branch is unpushed by definition.

## 2. Archive the paperwork

The ticket, the plan, the reviews and the pull request body are the record of why the change
looks the way it does. Keep them:

```bash
mkdir -p <kitchen>/archive
mv <station_dir> <kitchen>/archive/<slug>-$(date +%Y%m%d-%H%M%S)
```

## 3. Remove the station

Which command depends on the `workspace` field, and both cases are normal:

```bash
# workspace is a real id: closes the workspace, kills the pane and removes the worktree in one step
herdr worktree remove --workspace <workspace> --force
```

```bash
# workspace is "closed": there is no workspace to close, so remove the checkout directly
git -C <repo> worktree remove --force <worktree>
```

A `closed` workspace is the state `/brigade` reports as `no session` - the chef exited or the pane
was closed. `herdr worktree remove` needs a workspace id and has none to take, so the git path is
the one that works. Removing the checkout leaves no herdr workspace behind, because there was none.

## 4. The branch is a separate decision

Neither teardown path deletes the branch, and that is the right default: it may be under review or
already pushed. Step 1 already answered whether it is merged, in the `merged` field, so delete it
only when the user asks:

```bash
git -C <repo> branch -d <branch>   # -d, never -D
```

Use `-d`, never `-D`. If `-d` refuses, the branch has unmerged commits and the user needs to know
that rather than have it forced away. Note that `merged` was computed against `baseref`, so in a
repository with no `origin` it answers "merged into the local base", which is the most that can be
known there - say which one you checked.

## 5. Report

Say what was removed and what survived: the archived station directory, and the branch if it is
still there. If a pull request is open for that branch, remind the user it is still open.
