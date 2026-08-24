---
name: github-issue-response
description: Respond to a GitHub issue.
disable-model-invocation: true
---

# GitHub issue response

Use `gh` for GitHub operations. Resolve the issue from its URL, repository and number, or the current repository context.

## Workflow

1. Read the issue title, body, metadata, and comments. Identify the requested outcome, constraints, prior commitments, and unresolved questions.
2. Read the user's instructions about what to do and how the response should sound. Preserve their intent, tone, language, and level of detail.
3. Explore the repository when code, configuration, tests, history, or documentation is needed to answer accurately. Do not claim that something works, was changed, or was tested without evidence.
4. Compose a clear, self-contained response grounded in the issue and repository context. If the request is ambiguous, ask a focused clarifying question in the comment.
5. Before posting, show the complete proposed comment and confirm the target issue unless the user explicitly authorized posting.
6. Post the approved response with `gh issue comment`, preferably using `--body-file` for multiline Markdown.

## Boundaries

- Do not expose credentials or run commands that print tokens.
- Do not close the issue, edit existing comments, or change labels, assignees, or milestones unless separately requested.
- Do not make unrelated code or GitHub changes.
- If posting fails, report the error and do not retry the mutation automatically.
