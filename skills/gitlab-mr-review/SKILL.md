---
name: gitlab-mr-review
description: Review an existing GitLab merge request
disable-model-invocation: true
---

# GitLab MR Review

## Preconditions

1. Confirm `origin` is a GitLab remote for a zynca repository (`git config --get remote.origin.url`) and that `glab` is authenticated (`glab auth status`). Stop with a clear error if either check fails. A dirty checkout is fine. When the sandbox blocks a command — a missing keyring credential, `git fetch`, or `git worktree` — retry that command with sandbox escalation.

## Resolve the MR

2. Resolve the iid from a GitLab URL, an iid, or `glab mr view` on the current branch. Read that branch from the user's checkout, before the review worktree exists.

3. Read `glab mr view <mr-iid> --output json | jq '{source_branch, target_branch, title, description, diff_refs}'`. The review diff and every comment pin use `diff_refs.base_sha` and `diff_refs.head_sha`.

## Review worktree

4. Detach a sibling worktree at `head_sha`, so the user's checkout keeps its branch.

```bash
git fetch origin refs/merge-requests/<mr-iid>/head
git fetch origin <target_branch>   # base_sha must exist locally
REPO=$(git rev-parse --show-toplevel)
WORKTREE="$(dirname "$REPO")/$(basename "$REPO")-mr-<mr-iid>"
git worktree add --detach "$WORKTREE" <diff_refs.head_sha>
```

If the merge-request ref is missing, `git fetch origin <source_branch>` and add the worktree at `head_sha`. Reuse `$WORKTREE` when it is already a worktree whose HEAD equals `head_sha`. Otherwise `git worktree remove --force` it, `git worktree prune` if the path stays registered, and create it again.

Done when `git -C "$WORKTREE" rev-parse HEAD` equals `diff_refs.head_sha`, `base_sha` resolves, and `git -C "$WORKTREE" diff <base_sha>...HEAD` is non-empty. That diff is the review input. Run every later `git` command and file read in this worktree.

## Spec

5. If the MR names `PRO-####`, start the spec with `acli jira workitem view PRO-#### --fields "summary,description"` — those two fields — then the MR title and description. With no ticket, the spec is the MR title and description. When that text names no behaviour to check, the spec is `none`.

## Review

6. Read [`../detailed-code-review/SKILL.md`](../detailed-code-review/SKILL.md) and follow it as the caller:

- **Diff command**: `git -C <worktree> diff <base_sha>...HEAD`
- **Commit list command**: `git -C <worktree> log <base_sha>..HEAD --oneline`
- **Review root**: the worktree's absolute path, prefixed onto every changed-file path
- **Spec**: step 5
- **Overrides**: any the user gave (`+design`, `-data`, ...)

Show the report unchanged.

## Preview, then post

7. With zero findings, tell the user.

8. Otherwise show each comment you will post, then wait. An explicit yes creates the notes. Any other reply leaves them unposted. For each comment show the path, the pin (`line`, `old_line`, or unpinned), the reviewer, and the exact `-m` text. One finding per `-m`: its report section, severity, the finding, and the suggested fix. If the MR already has unresolved discussions, say so in that preview.

`glab mr note create <mr-iid>`:

- `line N` → `--file <path> --line N`; `line N-M` → `--line N:M`
- `old_line N` → `--file <path> --old-line N`
- a path and no line → `--file <path>`
- `unpinned` → omit `--file` and both line flags

A failed create stops the posting. Report the error and keep the pin the review gave.

## Finish

9. Remove the worktree when the review is done or cancelled.

```bash
git worktree remove --force "$WORKTREE"
```
