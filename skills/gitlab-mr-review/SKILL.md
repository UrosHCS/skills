---
name: gitlab-mr-review
description: Review an existing GitLab merge request against the SHAs GitLab stored for that MR, using the detailed-code-review skill and an optional Jira spec, then preview and post inline comments. Use when asked to review a GitLab MR by URL, iid, or the MR for the current branch. Do not use for GitHub pull requests.
disable-model-invocation: true
---

# GitLab MR Review

Reviews the merge request's stored diff (`git diff <base_sha>...HEAD` once HEAD is `head_sha`) with the `detailed-code-review` skill, which runs its parallel reviewers and returns line-pinned findings. Findings are previewed, then posted as inline MR comments only after the user confirms.

## Preconditions

Remember the current branch name, or the detached SHA if HEAD is detached. Restore that ref after the review, and if you stop early after checking out the MR, so the worktree is not left detached.

1. Confirm `origin` is a GitLab remote and a zynca GitLab repository (`git config --get remote.origin.url`). Confirm `glab` is authenticated (`glab auth status` - if it reports a missing token/keyring credential inside the sandbox, retry it with sandbox escalation). Confirm the worktree is clean (`git status --porcelain`). If any check fails, respond with a clear error and do not continue.

## Resolve the MR

2. Resolve the MR iid from a GitLab URL, an iid, or the open MR for the current branch (`glab mr view`).

3. Use `glab mr view <mr-iid> --output json | jq '{source_branch, target_branch, title, description, sha, diff_refs}'` to get the MR details. Keep `diff_refs.base_sha` and `diff_refs.head_sha`. Review and comment placement follow those SHAs, not local branch names.

## Fetch the MR tree

4. Fetch the MR head and detach onto GitLab's `head_sha`. Do not `git pull`. Do not check out the source or target branch names.

```bash
git fetch origin refs/merge-requests/<mr-iid>/head
git fetch origin <target_branch>   # so base_sha exists locally
git checkout --detach <diff_refs.head_sha>
```

If the merge-request ref is missing, `git fetch origin <source_branch>` instead, then detach onto `head_sha`. Confirm `git rev-parse HEAD` equals `diff_refs.head_sha`. Do not review unpushed local commits that are not in the MR.

Do not use `glab mr diff` as the review input. Once HEAD is `head_sha` and the fixed point is `base_sha`, `git diff <base_sha>...HEAD` is the same three-dot diff GitLab stored for this MR version. Confirm `base_sha` resolves and that diff is non-empty before starting the review.

## Spec source

5. If the MR details contain a jira ticket id in the format `PRO-####`, use `acli jira workitem view PRO-#### --fields "summary,description"` to get the ticket info. Do not follow links from the description or try to view images. Spec's primary source is the jira ticket info (if any); MR title and description are secondary. If both are thin, there is no spec: pass "none" rather than fabricating one.

## Review

6. Read the `detailed-code-review` skill (`../detailed-code-review/SKILL.md`, next to this skill's directory) and follow it as a calling skill, with these inputs:

- **Diff command**: `git diff <base_sha>...HEAD`.
- **Commit list command**: `git log <base_sha>..HEAD --oneline`.
- **Spec**: the Jira ticket summary and description, followed by the MR title and description, as spec text; or "none".
- **Overrides**: pass on any the user gave (`+design`, `-data`, ...).

Because these inputs are supplied, the review does not ask the user anything. Show the user its report as returned.

Restore the original branch (or detached SHA) now that the review is done.

## Preview then post

If the review has zero findings, say so and do not post.

Otherwise pick the findings to post: 🔴 and 🟡 only, most severe first, at most 8 inline comments. Put any remaining 🔴 and 🟡 findings in one short overview thread. Skip ⚪ findings and praise.

Show the proposed GitLab comments before posting: path, `line` / `old_line` or unpinned, reviewer, and the exact `-m` text. Each `-m` is one finding, labelled with its reviewer (the report section it came from) and severity, and includes the finding and its suggested fix. Do not paste the whole review into one message. If the MR already has unresolved discussions, mention that in the preview so a second run is an explicit choice. Do not use `--unique` (it cannot combine with `--file`).

Do not post until the user explicitly confirms.

Then post each selected finding as its own inline comment. A `line N` pin maps to `--line N`; a range `line N-M` maps to `--line N:M`; an `old_line N` pin maps to `--old-line N` (removed lines only). Omit both line flags for a file-level comment. If a finding is `unpinned`, post it as a general comment on the MR.

Post with `glab mr note create` (not `glab mr comment add`):

```bash
# added or unchanged line on the new side
glab mr note create <mr-iid> --file path/to/file.go --line 42 -m "…"

# removed line on the old side
glab mr note create <mr-iid> --file path/to/file.go --old-line 7 -m "…"

# optional: one short overview thread, not a paste of every finding
glab mr note create <mr-iid> -m "…"
```

After a failed note create (line not in the latest diff, rename path, stale SHA), stop and report. Do not retry with guessed line numbers.
