# Merge request

Describe the **committed** changes between `git merge-base origin/develop HEAD` and `HEAD` for a reviewer. A request to make or update an MR is approval to publish; a request for only a draft ends after step 4 with the draft shown.

## 1. Clean worktree

`git status --porcelain` is empty. If it is dirty, say so and offer a draft of the committed changes only, without verification or publishing.

## 2. Verification

When the diff touches `packages/ats/` or `packages/backend/`, the evidence for Testing is `node private/mr-verify.mjs --status`:

- `VALID`: use its `PASSED:` / `FAILED:` lines verbatim.
- `MISSING` or `STALE`: run `node private/mr-verify.mjs` once (several minutes; a non-zero exit is fine), then `--status` again.
- A `FAILED:` line blaming a busy port means that suite never ran: list it under _Not run_ and tell the user.
- A `STOPPED:` line means the verifier stopped the user's dev server: tell them to restart it.

## 3. Title

The commit convention from `SKILL.md`, with a specific lower-case summary.

- `feat` for a user-visible addition, `fix` for a defect; another conventional type only when obvious, otherwise ask.
- The ticket comes from the branch name only (`PRO-<digits>`, uppercased). No ticket there → omit the ` - PRO-####` suffix. Several → ask.

## 4. Description

Read the diff and the code behind it; every claim traces to the diff. Write the description with /pr, plus:

## 5. Publish

1. Push the branch, setting upstream if needed. A rejected push stops here; report it to the user rather than forcing.
2. Find the open MR with `glab mr view <branch> --output json` (this `glab` has no `--fields`).
3. `glab mr create` or `glab mr update` with the title, description, assignee (the authenticated `glab` user), and reviewer: `veljko6` when every change is in `packages/ats/`, otherwise `aleksandar.northprim446`.
4. Report the MR URL, whether it was created or updated, the title, assignee, and reviewer. After an update, include the description it replaced.
