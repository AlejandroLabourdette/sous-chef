---
name: brigade
description: Show the live status of every station in the brigade - slug, branch, herdr lifecycle state, workspace to switch to, and how far ahead of the base branch each one is. Use when the user asks how the stations are doing, what is running, which one needs them, or for a status of the parallel work.
---

# /brigade

Report the state of every station. **Compose this from herdr and git every time.** There is no
stored index, on purpose: an index would drift and then lie about which stations exist.

## Gather

```bash
REPO="$(git rev-parse --show-toplevel)"
BASE="$(git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##')"
[ -n "$BASE" ] || BASE="$(git rev-parse --abbrev-ref HEAD)"
KITCHEN="$HOME/.sous-chef/$(basename "$REPO")-$(printf '%s' "$REPO" | shasum | cut -c1-8)"

jq -n \
  --argjson w "$(herdr worktree list --cwd "$REPO")" \
  --argjson a "$(herdr agent list)" \
  --argjson k "$(ls "$KITCHEN" 2>/dev/null | grep -v '^archive$' | jq -Rs 'split("\n") | map(select(length > 0))')" '
  ($a.result.agents // []) as $ag
  | [ $w.result.worktrees[]
      | . as $wt
      | (($wt.branch // "") | split("/") | last) as $slug
      | select($k | index($slug))
      | ([$ag[] | select(.cwd == $wt.path)] | first) as $g
      | { slug:    $slug,
          branch:  $wt.branch,
          status: ($g.agent_status // "no session"),
          ws:     ($wt.open_workspace_id // "closed"),
          path:    $wt.path } ]'
```

A station is a worktree whose branch ends in a slug you have a ticket for. That is the join, and
it is why the branch prefix can be anything: the station directories under `$KITCHEN` already
exist as durable intent, so no index has to be invented to recognise your own worktrees. Filtering
on the branch prefix instead would sweep in the user's own `feat/*` branches, which follow the
same convention.

Then, per station, the git side, on the branch the join just gave you:

```bash
git -C "$REPO" rev-list --left-right --count "$BASE...$BRANCH"   # behind <tab> ahead
git -C "$WT" status --porcelain | head -1                        # non-empty = dirty
```

## Render

One line per station, ordered by urgency: `blocked` first, then `idle` and `done`, then
`working`. A station nobody is waiting on is the least interesting line on the screen.

```
STATION    STATUS    AHEAD  TREE    WORKSPACE  BRANCH
auth       blocked   3      clean   w4         refactor/auth
dark-mode  working   7      dirty   w5         feat/dark-mode
flaky-test done      2      clean   w6         fix/flaky-test
```

Read the statuses correctly, and translate them for the user rather than echoing herdr's
vocabulary:

| herdr status | What to tell the user |
| --- | --- |
| `working` | busy, nothing to do |
| `blocked` | **waiting on you**, in its own tab, at a question or approval |
| `idle` | finished its turn and ready for input |
| `done` | finished while you were not looking |
| `unknown` | a session is there but its state is unclear, which is **not** proof it finished |
| `no session` | the worktree exists but no chef is running in it |

Close with what to do next: which stations need the user, and how to get there
(`herdr workspace focus <ws>`, or the herdr workspace picker). Offer to focus one, but do not
focus anything unasked.

## Edge cases worth reporting instead of hiding

- Two rows with the same slug: two worktrees have branches ending in `/<slug>`, usually a station
  next to a branch of the user's own. Report both branches and say which one is the station - it
  is the worktree whose `path` matches that agent's `cwd`. Never pick one silently; `/plate` and
  `/86` cannot act on an ambiguous slug at all.
- A worktree with `no session`: the chef exited or the pane was closed. Offer to restart a chef
  in it, or to `/86` it.
- A dirty tree on a station the user believes is finished: say so before any talk of `/pass` or
  `/plate`, because uncommitted work is invisible to a branch review.
