---
name: pass
description: Send a station through the pass. Runs an independent code review of that station's branch as a background subagent, writes a numbered report, and delivers it into the station's own tab for the user to triage. Use when the user says a station is ready, is happy with its work, or asks for a review of a station's changes.
argument-hint: <slug>
---

# /pass

The review gate a station goes through before it can be plated. Three properties define it:
**the user asked for it** (when a chef pings you as ready, relay it in the shape that "Relaying a
station's ping" fixes in the `sous-chef` skill, and stop there), **the reviewer is independent**
of the chef that wrote the code, and **you do not block** while it runs.

## 1. Resolve and check, in one call

Run the resolve preamble from the `sous-chef` skill and this tail as a **single** `Bash` call. The
checks depend on `$WT` and `$BASEREF` being set in that same shell; split across calls they read
empty, and `git -C "" status` then reports on your own checkout and comes back clean - a dirty
station would sail through the one guard meant to catch it.

```bash
if [ "$MATCHES" = 1 ]; then
  BRANCH="$(printf '%s' "$ROWS" | jq -r '.[0].branch')"
  WT="$(printf '%s' "$ROWS" | jq -r '.[0].path')"
  WS="$(printf '%s' "$ROWS" | jq -r '.[0].open_workspace_id // "closed"')"
  DIRTY="$(git -C "$WT" status --porcelain | head -1)"
  AHEAD="$(git -C "$REPO" rev-list --count "$BASEREF..$BRANCH")"
  NEXT="$(( $(find "$KITCHEN/$SLUG" -maxdepth 1 -name 'review-*.md' 2>/dev/null | wc -l) + 1 ))"
fi
jq -n --argjson m "$MATCHES" --argjson rows "$ROWS" \
      --arg b "${BRANCH:-}" --arg w "${WT:-}" --arg ws "${WS:-}" --arg br "$BASEREF" \
      --arg k "$KITCHEN/$SLUG" --arg d "${DIRTY:-}" --arg a "${AHEAD:-}" --arg n "${NEXT:-}" \
  '{matches: $m, all_matches: [$rows[].branch], branch: $b, worktree: $w, workspace: $ws,
    baseref: $br, station_dir: $k, dirty: ($d != ""), ahead: $a, next_review: $n}'
```

Read the object before doing anything with it:

- **`matches` is not 1** - stop, as the `sous-chef` skill describes. `all_matches` names the
  branches to report.
- **`dirty` is true** - uncommitted work the review would not see. Tell the user and ask the
  station to commit first.
- **`ahead` is `0`** - there is nothing to review. Say so rather than dispatching a reviewer at an
  empty diff.
- **`workspace` is `closed`** - the chef is gone. The branch is still reviewable, but step 3 has
  nowhere to deliver the report; say so and offer to restart the chef first.

`next_review` is the report number for step 2, and `station_dir` is where it goes.

## 2. Dispatch the reviewer in the background

Spawn a background subagent with this shape of brief, filling the placeholders from the object
step 1 printed:

> Review the branch `<branch>` of the repository at `<REPO>` against its base `<baseref>`.
>
> Work from the main repository, not from the worktree. The station branch is a ref in the same
> object store, so `git -C <REPO> diff <baseref>...<branch>` gives you the full change and
> `git -C <REPO> show <branch>:<path>` gives you any file at that branch. You do not need to enter
> the worktree.
>
> The ticket that produced this work is at `<station_dir>/ticket.md`. Read it first: a change
> that is clean but does not satisfy the ticket is the most important finding you can make.
> If earlier reports exist at `<station_dir>/review-*.md`, read them too and do not repeat
> findings the user already declined.
>
> If `<station_dir>/plan.md` exists, read it as well. It is the phase plan the user approved,
> and the branch is meant to be that plan with one commit per phase. A commit that bundles two
> phases, a phase that never landed, or a change that is in the diff but nowhere in the plan is a
> finding in its own right.
>
> Use the `code-review` skill against this branch target for the analysis.
>
> Write your findings to `<station_dir>/review-<next_review>.md`. Order them most severe first, and for
> each one give the file and line, what is wrong, and the concrete failure it causes. Separate
> correctness bugs from simplification and reuse suggestions, and mark anything you are not
> confident about as uncertain rather than dropping it or overstating it. If the branch is clean,
> say that plainly in the file instead of manufacturing findings.
>
> Return two numbers and nothing else: how many correctness findings, and how many quality
> findings. The report carries the detail, and the orchestrator relays only the counts.

Tell the user the review is running and that you are free meanwhile. Then stop.

## 3. Deliver it into the station's tab, on completion

When the background subagent reports back - you do not wait for it, the result arrives on its own -
deliver the report into the station:

```bash
herdr agent prompt <slug> "Review <n> is at <station_dir>/review-<n>.md. Read it, summarize the findings for the user here, and wait for their direction on which to implement. Do not implement anything on your own initiative."
herdr notification show <slug> --body "review <n> ready" --sound done
```

This is a separate call from step 1, so nothing is still set: substitute the literal values.

Then tell the user one line: the two counts, which review landed, and the workspace to switch to.
The counts are the one thing a relay may carry beyond the news itself, because a count is not a
finding - it says whether there is anything to triage without deciding any of it, and a clean
review has to be able to say so without costing a tab switch.

**Nothing past the counts.** The chef is about to summarize that same report in its own tab, with
the code in front of it, and that is where the user picks what gets implemented.

## 4. Repeat as needed

`/pass` is repeatable: after the chef implements the findings the user picked, pass again and the
reviewer writes `review-2.md`, aware of what came before. Keep passing until the user says they
are satisfied, then `/plate`.
