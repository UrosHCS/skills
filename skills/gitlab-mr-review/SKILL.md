---
name: gitlab-mr-review
description: Review an existing GitLab merge request against its target branch using the advanced-review skill and optional Jira spec.
disable-model-invocation: true
---

1. Run `git config --get remote.origin.url` and make sure that the origin is a zynca gitlab repository. Run git status to make sure there are no uncommitted changes. If either check fails respond with a clear error message and do not continue.

2. Resolve the MR iid from the user's instructions.

3. Use `glab mr view <mr-iid> --output json | jq '{source_branch, target_branch, title, description, sha, diff_refs}'` to get the MR details. Keep `diff_refs.base_sha`, `diff_refs.head_sha`, and `diff_refs.start_sha`; comment placement is relative to those SHAs, not to whatever the local branch names happen to resolve to.

4. Run `git fetch`, `git checkout <target_branch>`, `git pull`, `git checkout <source_branch>`, and `git pull` so that the review sub-agents can explore the codebase. Then confirm `git rev-parse HEAD` equals `diff_refs.head_sha`. If it does not, check out `diff_refs.head_sha` (detached is fine) and review that commit. Do not review unpushed local commits that are not in the MR.

Do not use `glab mr diff` as the review input. `advanced-review` needs a worktree, and once HEAD is `head_sha` and the fixed point is `base_sha`, `git diff <base_sha>...HEAD` is the same three-dot diff GitLab stored for this MR version.

5. If the MR details contain a jira ticket id in the format `PRO-####`, use `acli jira workitem view PRO-#### --fields "summary,description"` to get the ticket info. Do not follow links from the description or try to view images. Only rely on the summary and description if they help at all.

6. Use the "advanced-review" skill to review the work, with these arguments:
- fixed point is `diff_refs.base_sha` (not the target branch name)
- spec's primary source is the jira ticket info (if any) and the MR details is secondary (if both are thin, tell `advanced-review` there is no spec rather than fabricating one)
- extra reporting requirement: every finding must include the file path as it appears in the diff (new path after renames) and the line it refers to. Use `line` for the new/source side, or `old_line` only for a removed line. Take those numbers from the diff hunk (`@@ -old +new @@` plus the `+`/`-`/context lines), not from reading the file in isolation. If a finding is not about a specific changed line (for example a missing requirement with no implementation site), say that explicitly instead of guessing a line.

7. Post each review finding as its own inline comment on the MR diff. Do not dump the whole review into one `--body`. Use the path and `line` / `old_line` returned by `advanced-review`. `--line` is the new/source-side number; `--old-line` is only for a removed line. Use `--line 10:15` for a contiguous range. Omit both line flags for a file-level comment. If a finding has no pin, post it as a general comment on the MR.

Post with `glab mr note create` (not `glab mr comment add`):

```bash
# added or unchanged line on the new side
glab mr note create <mr-iid> --file path/to/file.go --line 42 -m "…"

# removed line on the old side
glab mr note create <mr-iid> --file path/to/file.go --old-line 7 -m "…"

# optional: one short overview thread, not a paste of every finding
glab mr note create <mr-iid> -m "…"
```
