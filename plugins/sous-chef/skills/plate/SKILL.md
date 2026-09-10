---
name: plate
description: Plate a station. Verifies the branch is clean and mergeable, confirms with the user, then pushes it and opens a pull request against the base branch. Use when the user is satisfied with a station's work and wants to ship it, publish the branch, or open the PR.
argument-hint: <slug>
---

# /plate

The only verb that touches the remote, and the first outward-facing action in the flow. It
confirms before it acts.

## 1. Verify before promising anything, in one call

Run the resolve preamble from the `sous-chef` skill and this tail as a **single** `Bash` call.
`$WT`, `$BRANCH` and `$BASEREF` have to be live in the same shell as the checks that use them.

```bash
if [ "$MATCHES" = 1 ]; then
  BRANCH="$(printf '%s' "$ROWS" | jq -r '.[0].branch')"
  WT="$(printf '%s' "$ROWS" | jq -r '.[0].path')"
  DIRTY="$(git -C "$WT" status --porcelain | head -1)"
  git -C "$REPO" fetch origin --quiet 2>/dev/null
  AHEAD="$(git -C "$REPO" rev-list --count "$BASEREF..$BRANCH" 2>/dev/null)"
  git -C "$REPO" merge-base --is-ancestor "$BASEREF" "$BRANCH" 2>/dev/null; SYNC=$?
  REVIEWS="$(find "$KITCHEN/$SLUG" -maxdepth 1 -name 'review-*.md' 2>/dev/null | wc -l | tr -d ' ')"
  COMMITS="$(git -C "$REPO" log --oneline "$BASEREF..$BRANCH" 2>/dev/null)"
fi
jq -n --argjson m "$MATCHES" --argjson rows "$ROWS" \
      --arg b "${BRANCH:-}" --arg w "${WT:-}" --arg br "$BASEREF" --arg k "$KITCHEN/$SLUG" \
      --arg d "${DIRTY:-}" --arg a "${AHEAD:-}" --arg sy "${SYNC:-}" \
      --arg r "${REVIEWS:-0}" --arg c "${COMMITS:-}" \
  '{matches: $m, all_matches: [$rows[].branch], branch: $b, worktree: $w, baseref: $br,
    station_dir: $k, dirty: ($d != ""), ahead: $a, sync: $sy, reviews: $r,
    commits: ($c | split("\n") | map(select(length > 0)))}'
```

Read it before promising the user anything:

- **`matches` is not 1** - stop, as the `sous-chef` skill describes.
- **`dirty` is true** - uncommitted work. The push would ship something other than what the user
  looked at. Ask the station to commit first.
- **`ahead` is `0`** - nothing to ship.
- **`sync` is `0`** - the branch contains the base. Carry on.
- **`sync` is `1`** - the base has moved on and the branch is behind. Have the station rebase, not
  you: it is that chef's worktree and it has the context to resolve conflicts.

  ```bash
  herdr agent prompt <slug> "sous-chef: <base> has moved on. Rebase <branch> onto <baseref>, resolve any conflicts, make sure the tests still pass, and report back when the branch is clean."
  ```

  **Then stop.** The rebase happens in the station's own time and the branch is not shippable until
  it lands. Tell the user what you asked for and that they should `/plate` again once the station
  reports back. Do not carry on into step 2.
- **`sync` is `128`** - git could not compare, which is a different answer from "behind". The usual
  cause is a repository with no `origin`, where `baseref` fell back to a local branch. Report that
  you cannot tell whether the branch is current and let the user decide; do not send the station on
  a rebase onto a ref that may not exist.
- **`reviews` is `0`** - the station never went through the pass. Say so and ask whether to `/pass`
  first. Do not refuse to plate; it is the user's call.

`commits` is the list step 2 shows the user.

## 2. Confirm, showing the real thing

Show what will happen before it happens: the branch, the base, the commit count, a one-line
summary of each commit, and the pull request title and body you intend to create. Then ask.

Do not push on an implied yes. "Ship it" is the request; the confirmation is on the concrete plan
you just showed.

## 3. Push and open the pull request

First write the body to `<station_dir>/pr-body.md`. It is the fourth file a station keeps, and it is
durable intent like the others: it records what was shipped, so `/86` archives it alongside the
ticket, the plan and the reviews.

Build it from what you already have, and keep it short:

- What the ticket asked for, in a sentence or two, from `ticket.md`.
- What actually changed, at the level of behaviour. Build this from the phases in `plan.md`: they
  map one to one onto the `commits` step 1 listed.
- Anything the reviews raised that was consciously left undone, and why.

No attribution or co-author trailers, for you or for the chef.

Then push. This is a new `Bash` call, so nothing from step 1 is still set: **substitute the literal
values** from the object it printed rather than writing `$WT` or `$BRANCH` and hoping.

```bash
git -C <worktree> push -u origin <branch>
gh pr create --repo "$(gh repo view --json nameWithOwner -q .nameWithOwner)" \
  --base <base> --head <branch> \
  --title "<title>" --body-file <station_dir>/pr-body.md
```

`--base` takes `base`, the branch name, not `baseref`: GitHub wants the branch the pull request
targets, not a local ref. If `baseref` fell back because there is no `origin`, there is nowhere to
push and no pull request to open - say so instead of trying.

## 4. Do not merge

Report the pull request URL and stop. Merging is the user's decision, made on GitHub with CI in
front of them. Leave the station open: review comments may come back to it. Offer `/86 <slug>`
only once the pull request is merged or closed.

## If the push fails

Report the actual git or gh error verbatim. The usual causes: no `origin` remote, `gh` not
authenticated (`gh auth login`), or a protected branch or missing fork, which is a repository
policy question for the user.

Never retry a failed push with different flags, and never force push.
