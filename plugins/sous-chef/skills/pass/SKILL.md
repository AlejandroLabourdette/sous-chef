---
name: pass
description: Send a station through the pass. Runs an independent code review of that station's branch as a background subagent, writes a numbered report, and delivers it into the station's own tab for the user to triage. Use when the user says a station is ready, is happy with its work, or asks for a review of a station's changes.
argument-hint: <slug>
---

# /pass

In a kitchen, the pass is where every dish is checked before it leaves. Here it is the review
gate a station goes through before it can be plated.

Two properties define this verb:

1. **The review is independent.** A separate reviewer looks at the branch, not the chef that
   wrote it. An agent reviewing its own work finds far less.
2. **You do not block.** The review runs as a background subagent. You dispatch it and go
   straight back to the user, who can fire another ticket or pass another station while it runs.

## 1. Check the branch is worth reviewing

```bash
git -C "$WT" status --porcelain          # must be empty
git -C "$REPO" rev-list --count "$BASE..sous-chef/$SLUG"   # must be > 0
```

A dirty tree means uncommitted work that the review would not see. Tell the user and ask the
station to commit first, rather than reviewing an incomplete picture.

## 2. Dispatch the reviewer in the background

Pick the report number from what already exists:

```bash
N=$(( $(ls "$KITCHEN/$SLUG"/review-*.md 2>/dev/null | wc -l) + 1 ))
```

Then spawn a background subagent. Give it this shape of brief:

> Review the branch `sous-chef/<slug>` of the repository at `<REPO>` against its base `<BASE>`.
>
> Work from the main repository, not from the worktree. The station branch is a ref in the same
> object store, so `git -C <REPO> diff <BASE>...sous-chef/<slug>` gives you the full change and
> `git -C <REPO> show sous-chef/<slug>:<path>` gives you any file at that branch. You do not need
> to enter the worktree.
>
> The ticket that produced this work is at `<KITCHEN>/<slug>/ticket.md`. Read it first: a change
> that is clean but does not satisfy the ticket is the most important finding you can make.
> If earlier reports exist at `<KITCHEN>/<slug>/review-*.md`, read them too and do not repeat
> findings the user already declined.
>
> Use the `code-review` skill against this branch target for the analysis.
>
> Write your findings to `<KITCHEN>/<slug>/review-<N>.md`. Order them most severe first, and for
> each one give the file and line, what is wrong, and the concrete failure it causes. Separate
> correctness bugs from simplification and reuse suggestions, and mark anything you are not
> confident about as uncertain rather than dropping it or overstating it. If the branch is clean,
> say that plainly in the file instead of manufacturing findings.
>
> Return a three line summary: how many correctness findings, how many quality findings, and the
> single most important one.

Reuse the existing `code-review` skill rather than inventing a reviewer. It already handles
severity, verification and false-positive filtering.

Tell the user the review is running, and that you are free in the meantime. Then stop.

## 3. Deliver it into the station's tab, on completion

When the background task reports back:

```bash
herdr agent prompt "$SLUG" "Review $N is at $KITCHEN/$SLUG/review-$N.md. Read it, summarize the findings for the user here, and wait for their direction on which to implement. Do not implement anything on your own initiative."
herdr notification show "$SLUG" --body "review $N ready" --sound done
```

Then tell the user, in one or two lines: the headline of the review and which station's tab it
landed in. The triage itself happens there, with the chef that knows the code.

## 4. Repeat as needed

`/pass` is repeatable. After the chef implements the findings the user picked, pass it again and
the reviewer will write `review-2.md`, aware of what came before. Keep passing until the user
says they are satisfied, then `/plate`.
