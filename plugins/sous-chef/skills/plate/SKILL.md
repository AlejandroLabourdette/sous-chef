---
name: plate
description: Plate a station. Verifies the branch is clean and mergeable, confirms with the user, then pushes it and opens a pull request against the base branch. Use when the user is satisfied with a station's work and wants to ship it, publish the branch, or open the PR.
argument-hint: <slug>
---

# /plate

The only verb that touches the remote, and the first outward-facing action in the flow. It
confirms before it acts.

## 1. Verify before promising anything

Resolve the station first, as described in the `sous-chef` skill: `$BRANCH` and `$WT` come from
herdr, never from the slug.

```bash
git -C "$WT" status --porcelain                              # must be empty
git -C "$REPO" rev-list --count "$BASE..$BRANCH"             # must be > 0
git -C "$REPO" fetch origin --quiet
git -C "$REPO" merge-base --is-ancestor "origin/$BASE" "$BRANCH"   # 0 = up to date with base
```

If the branch is behind the base, have the station rebase, not you: it is that chef's worktree
and it has the context to resolve conflicts.

```bash
herdr agent prompt "$SLUG" "sous-chef: $BASE has moved on. Rebase $BRANCH onto origin/$BASE, resolve any conflicts, make sure the tests still pass, and report back when the branch is clean."
```

Then check whether the station went through the pass at all:

```bash
ls "$KITCHEN/$SLUG"/review-*.md 2>/dev/null
```

If there is no review, say so and ask whether to `/pass` first. Do not refuse to plate; it is the
user's call.

## 2. Confirm, showing the real thing

Show what will happen before it happens: the branch, the base, the commit count, a one-line
summary of each commit, and the pull request title and body you intend to create. Then ask.

Do not push on an implied yes. "Ship it" is the request; the confirmation is on the concrete plan
you just showed.

## 3. Push and open the pull request

```bash
git -C "$WT" push -u origin "$BRANCH"
gh pr create --repo "$(gh repo view --json nameWithOwner -q .nameWithOwner)" \
  --base "$BASE" --head "$BRANCH" \
  --title "<title>" --body-file "$KITCHEN/$SLUG/pr-body.md"
```

Build the body from what you already have, and keep it short:

- What the ticket asked for, in a sentence or two, from `ticket.md`.
- What actually changed, at the level of behaviour. Build this from the phases in `plan.md`: they
  map one to one onto the commits you just listed.
- Anything the reviews raised that was consciously left undone, and why.

No attribution or co-author trailers, for you or for the chef.

## 4. Do not merge

Report the pull request URL and stop. Merging is the user's decision, made on GitHub with CI in
front of them. Leave the station open: review comments may come back to it. Offer `/86 <slug>`
only once the pull request is merged or closed.

## If the push fails

Report the actual git or gh error verbatim. The usual causes: no `origin` remote, `gh` not
authenticated (`gh auth login`), or a protected branch or missing fork, which is a repository
policy question for the user.

Never retry a failed push with different flags, and never force push.
