---
name: gitlab-mr-review
description: Review an existing GitLab merge request against its target branch using the advanced-review skill and optional Jira spec.
disable-model-invocation: true
---

1. Run `git config --get remote.origin.url` and make sure that the origin is a zynca gitlab repository. Run git status to make sure there are no uncommitted changes. If either check fails respond with a clear error message and do not continue.

2. Resolve the MR id from the user's instructions.

3. Use `glab mr view 6056 --output json | jq '{source_branch, target_branch, title, description}'` to get the MR details.

4. Run `git fetch`, `git checkout <target_branch>`, `git pull`, `git checkout <source_branch>`, and `git pull` so that the review sub-agents can explore the codebase.

5. If the MR details contain a jira ticket id in the format `PRO-####`, use `acli jira workitem view PRO-#### --fields "summary,description"` to get the ticket info. Do not follow links from the description or try to view images. Only rely on the summary and description if they help at all.

6. Use the "advanced-review" skill to review the work:
- fixed point is the MR's target branch
- spec's primary source is the jira ticket info (if any) and the MR details is secondary (if both are thin, tell `advanced-review` there is no spec rather than fabricating one)

7. Post comments on the MR using `glab mr comment add 6056 --body "Review comments"`