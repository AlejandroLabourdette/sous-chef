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

jq -n \
  --argjson w "$(herdr worktree list --cwd "$REPO")" \
  --argjson a "$(herdr agent list)" '
  ($a.result.agents // []) as $ag
  | [ $w.result.worktrees[]
      | select(.branch // "" | startswith("sous-chef/"))
      | . as $wt
      | ([$ag[] | select(.cwd == $wt.path)] | first) as $g
      | { slug:   ($wt.branch | sub("^sous-chef/"; "")),
          branch:  $wt.branch,
          status: ($g.agent_status // "no session"),
          ws:     ($wt.open_workspace_id // "closed"),
          path:    $wt.path } ]'
```

Then, per station, the git side:

```bash
git -C "$REPO" rev-list --left-right --count "$BASE...sous-chef/$SLUG"   # behind <tab> ahead
git -C "$WT" status --porcelain | head -1                                 # non-empty = dirty
```

## Render

One line per station, ordered by urgency: `blocked` first, then `idle` and `done`, then
`working`. A station nobody is waiting on is the least interesting line on the screen.

```
STATION    STATUS    AHEAD  TREE    WORKSPACE  BRANCH
auth       blocked   3      clean   w4         sous-chef/auth
dark-mode  working   7      dirty   w5         sous-chef/dark-mode
flaky-test done      2      clean   w6         sous-chef/flaky-test
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

- A branch `sous-chef/*` with no worktree: the station was torn down but the branch survived,
  which is normal after `/86`. Mention it as a leftover branch, not as a station.
- A worktree with `no session`: the chef exited or the pane was closed. Offer to restart a chef
  in it, or to `/86` it.
- A dirty tree on a station the user believes is finished: say so before any talk of `/pass` or
  `/plate`, because uncommitted work is invisible to a branch review.
