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

## 1. Check the branch is worth reviewing

Resolve the station first, as described in the `sous-chef` skill, so `$BRANCH` and `$WT` come
from herdr rather than from the slug. Keep `$WS` from that same lookup: step 3 relays the
workspace to switch to, and this is where that id comes from.

```bash
git -C "$WT" status --porcelain                   # must be empty
git -C "$REPO" rev-list --count "$BASE..$BRANCH"  # must be > 0
```

A dirty tree means uncommitted work the review would not see. Tell the user and ask the station
to commit first.

## 2. Dispatch the reviewer in the background

Pick the report number from what already exists:

```bash
N=$(( $(ls "$KITCHEN/$SLUG"/review-*.md 2>/dev/null | wc -l) + 1 ))
```

Then spawn a background subagent with this shape of brief:

> Review the branch `<BRANCH>` of the repository at `<REPO>` against its base `<BASE>`.
>
> Work from the main repository, not from the worktree. The station branch is a ref in the same
> object store, so `git -C <REPO> diff <BASE>...<BRANCH>` gives you the full change and
> `git -C <REPO> show <BRANCH>:<path>` gives you any file at that branch. You do not need to enter
> the worktree.
>
> The ticket that produced this work is at `<KITCHEN>/<slug>/ticket.md`. Read it first: a change
> that is clean but does not satisfy the ticket is the most important finding you can make.
> If earlier reports exist at `<KITCHEN>/<slug>/review-*.md`, read them too and do not repeat
> findings the user already declined.
>
> If `<KITCHEN>/<slug>/plan.md` exists, read it as well. It is the phase plan the user approved,
> and the branch is meant to be that plan with one commit per phase. A commit that bundles two
> phases, a phase that never landed, or a change that is in the diff but nowhere in the plan is a
> finding in its own right.
>
> Use the `code-review` skill against this branch target for the analysis.
>
> Write your findings to `<KITCHEN>/<slug>/review-<N>.md`. Order them most severe first, and for
> each one give the file and line, what is wrong, and the concrete failure it causes. Separate
> correctness bugs from simplification and reuse suggestions, and mark anything you are not
> confident about as uncertain rather than dropping it or overstating it. If the branch is clean,
> say that plainly in the file instead of manufacturing findings.
>
> Return two numbers and nothing else: how many correctness findings, and how many quality
> findings. The report carries the detail, and the orchestrator relays only the counts.

Tell the user the review is running and that you are free meanwhile. Then stop.

## 3. Deliver it into the station's tab, on completion

```bash
herdr agent prompt "$SLUG" "Review $N is at $KITCHEN/$SLUG/review-$N.md. Read it, summarize the findings for the user here, and wait for their direction on which to implement. Do not implement anything on your own initiative."
herdr notification show "$SLUG" --body "review $N ready" --sound done
```

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
