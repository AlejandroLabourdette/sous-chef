---
name: brigade
description: Show the live status of every station in the brigade - slug, branch, herdr lifecycle state, workspace to switch to, and how far ahead of the base branch each one is. Use when the user asks how the stations are doing, what is running, which one needs them, or for a status of the parallel work.
---

# /brigade

Report the state of every station. **Compose this from herdr and git every time.** There is no
stored index, on purpose: an index would drift and then lie about which stations exist.

## Gather

One `Bash` call. The per-station git facts have to be read with the worktree path live in the same
shell, so the loop runs inside the same invocation as the join - which also makes `/brigade` cost
two commands instead of two per station.

Run the resolve preamble from the `sous-chef` skill (with no `SLUG`; `$ROWS` and `$MATCHES` are not
used here), then:

```bash
herdr worktree list --cwd "$REPO" | jq -c --argjson a "$(herdr agent list)" \
  --argjson k "$(ls "$KITCHEN" 2>/dev/null | grep -v '^archive$' | jq -Rs 'split("\n") | map(select(length > 0))')" '
  ($a.result.agents // []) as $ag
  | [ .result.worktrees[]
      | . as $wt
      | (($wt.branch // "") | split("/") | last) as $slug
      | select($k | index($slug))
      | ([$ag[] | select(.cwd == $wt.path)] | first) as $g
      | { slug:   $slug,
          branch: $wt.branch,
          status: ($g.agent_status // "no session"),
          ws:     ($wt.open_workspace_id // "closed"),
          path:   $wt.path } ]' \
| jq -c '.[]' | while read -r row; do
    WT="$(printf '%s' "$row" | jq -r .path)"
    BR="$(printf '%s' "$row" | jq -r .branch)"
    COUNTS="$(git -C "$REPO" rev-list --left-right --count "$BASEREF...$BR" 2>/dev/null)"
    DIRTY="$(git -C "$WT" status --porcelain 2>/dev/null | head -1)"
    printf '%s' "$row" | jq -c --arg b "$(printf '%s' "$COUNTS" | cut -f1)" \
                              --arg a "$(printf '%s' "$COUNTS" | cut -f2)" \
                              --arg d "$DIRTY" \
      '. + {behind: $b, ahead: $a, dirty: ($d != "")}'
  done | jq -s .
```

A station is a worktree whose branch ends in a slug you have a ticket for. That join is why the
branch prefix can be anything: filtering on the prefix instead would sweep in the user's own
`feat/*` branches, which follow the same convention.

## Render

One line per station, ordered by urgency: `blocked` first, then `idle` and `done`, then
`working`.

```
STATION    STATUS    AHEAD  TREE    WORKSPACE  BRANCH
auth       blocked   3      clean   w4         refactor/auth
dark-mode  working   7      dirty   w5         feat/dark-mode
flaky-test done      2      clean   w6         fix/flaky-test
```

Translate the statuses for the user rather than echoing herdr's vocabulary:

| herdr status | What to tell the user |
| --- | --- |
| `working` | busy, nothing to do |
| `blocked` | **waiting on you**, in its own tab, at a question or approval |
| `idle` | finished its turn and ready for input |
| `done` | finished while you were not looking |
| `unknown` | a session is there but its state is unclear, which is **not** proof it finished |
| `no session` | the worktree exists but no chef is running in it |

Close with which stations need the user and how to get there (`herdr workspace focus <ws>`, or
the herdr workspace picker). Offer to focus one, but do not focus anything unasked.

## Edge cases worth reporting instead of hiding

- Two rows with the same slug: two worktrees have branches ending in `/<slug>`, usually a station
  next to a branch of the user's own. Report both branches and say which one is the station - it
  is the worktree whose `path` matches that agent's `cwd`. Never pick one silently; `/plate` and
  `/86` cannot act on an ambiguous slug at all.
- A worktree with `no session`: the chef exited or the pane was closed, and its `ws` reads
  `closed`. Offer to restart a chef in it, or to `/86` it - `/86` handles a closed workspace
  explicitly. Restarting takes the row's literal path and slug:

  ```bash
  herdr worktree open --cwd <repo> --path <path> --label <slug> --no-focus \
    | jq -r .result.root_pane.pane_id
  ```

  ```bash
  herdr agent start <slug> --kind claude --pane <pane> -- \
    --permission-mode plan -n <slug> --add-dir "$HOME/.sous-chef"
  herdr agent prompt <slug> "You are the chef-de-partie for station <slug>. Read <kitchen>/<slug>/ticket.md and plan.md, then invoke the chef-de-partie skill and follow it."
  ```

  The restarted chef reads the same ticket and plan, so it picks up from where the branch already
  is. It starts in plan mode like any station.
- A dirty tree on a station the user believes is finished: say so before any talk of `/pass` or
  `/plate`, because uncommitted work is invisible to a branch review.
