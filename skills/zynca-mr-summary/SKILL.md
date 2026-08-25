---
name: gitlab-mr-summary
description: Draft a precise GitLab merge-request summary for the current branch against `develop`, then, only on explicit confirmation, push the branch and create or update its GitLab MR.
disable-model-invocation: true
---

# GitLab MR Summary

Prepare a concise, reviewer-oriented description from the committed changes that will be in the merge request. Treat `develop` as the only target branch.

## Preconditions

1. Confirm the current directory is a Git worktree and the current branch is not `develop`.
2. Confirm `origin` is a GitLab remote and `glab` is installed and authenticated with access to it. Run `glab auth status --hostname gitlab.com`. If the current sandbox cannot read the operating-system keyring, repeat that read-only check with keyring access before treating the CLI as unauthenticated. Ask the user to authenticate with `glab auth login --hostname gitlab.com` only when that retry also fails; do not attempt to install it.
3. Check for uncommitted changes with `git status --porcelain`. Do not publish while the worktree is dirty, because those changes cannot be part of the MR. State this clearly and offer to draft a summary of committed changes only.
4. Fetch `origin/develop`, then calculate the merge base of `origin/develop` and `HEAD`. Use that merge base for every diff and commit query. Do not compare only with `HEAD~1`.
5. Determine whether the committed merge-base diff contains `packages/ats/` or `packages/backend/`. When it does, the worktree is clean, and `./private/mr-verify.mjs` exists, run `node ./private/mr-verify.mjs --status`. If it begins with `VALID`, use those compact status lines and do not read `./private/mr-verification.json`. If it begins with `MISSING` (no prior log) or `STALE` (the log belongs to a different source revision or has incomplete checks), run `node ./private/mr-verify.mjs` once, then run `--status` again. If verification exits non-zero, still use the refreshed compact status to disclose failed checks. The verifier type-checks every affected package first; ATS then runs formatting, lint, unit, and headless E2E checks, while backend runs formatting, lint, and unit checks only. Do not run verification in a dirty worktree because it would test uncommitted code that cannot be attributed to the MR.

## Draft the MR title

Derive a concise title from the actual merge-base diff. Use this exact shape:

```text
<type>: <concise lower-case change summary> - PRO-####
```

Choose `feat` for a user-visible addition and `fix` for a defect correction. Use another conventional type only when it is clearly justified by the changes or the branch name; otherwise ask the user. Keep the change summary specific, lower-case, and without terminal punctuation.

Extract a ticket only from the current branch name, matching `PRO-<digits>` case-insensitively and rendering the ticket key in uppercase. Emit the plain ticket key so GitLab can autolink it. If no ticket appears in the branch name, omit the space, hyphen, and ticket entirely. Never infer a ticket from commits, the existing MR, the diff, or Jira. If the branch contains multiple distinct tickets, ask the user which ticket belongs in the title.

## Draft the MR summary

Inspect the merge-base diff, changed-file statistics, commit history, tests that the user actually ran, and relevant implementation details. For each area with touched code, inspect the nearest applicable `package.json` and its `scripts` to identify every test type the affected package supports. Identify tests newly added by the change and what they cover. Do not infer a change from a file name alone. Do not describe uncommitted changes, unrelated commits, or tests that were not run.

Produce concise Markdown in this form:

```md
<!-- codex:mr-summary:start -->
## Summary
- …

## Changes
- …

## Testing
- …

## Risk / rollout
- …

## Reviewer guide
- …
<!-- codex:mr-summary:end -->
```

Omit empty sections rather than adding filler. In Testing, attribute executed checks to the user (for example, `User ran: npm test`). For ATS changes, use only `node ./private/mr-verify.mjs --status` as verification evidence. When it starts with `VALID`, state `User ran:` and list its compact passed or failed results, including the failed command. Do not read the detailed JSON report; reject `MISSING` or `STALE` status output. Also list every test type supported by the affected package(s), as deduced from the relevant `package.json` scripts (for example, unit, integration, end-to-end, visual, or component tests); do not list a type unless a corresponding script supports it. Keep supported test types distinct from checks actually run, and include a `Not run` item when appropriate. When the change adds tests, mention the new tests and the behaviour they cover. Make the summary useful to a reviewer: explain behavioural changes, migrations, feature flags, security or privacy implications, and API or UI impact where they exist.

## Preview before publishing

Show the complete proposed Markdown and state:

- proposed MR title;
- source branch;
- target branch: `develop`;
- whether an open MR exists for the current branch;
- whether the next action will create an MR or update one.
- the authenticated GitLab user who will be assigned;
- the default reviewers: every active, non-bot project member except the authenticated user and `danzync`, deduplicated by user ID.

Do not push, create an MR, or update an MR until the user explicitly asks to publish the displayed draft. Requests such as "summarize", "prepare", or "draft" are not publication approval.

## Publish after explicit confirmation

1. Push the current branch to `origin` without force-pushing. Set its upstream only when it has none.
2. Look up the open MR for the current branch with `glab mr view <branch> --output json`. Do not assume the installed `glab` version supports a `--fields` flag; parse the JSON response when specific fields are needed.
3. Resolve the authenticated GitLab user with `glab api user`. List active project members with `glab api 'projects/:id/members/all?per_page=100'`; dedupe by user ID and select every non-bot member except the authenticated user and `danzync`. This is the configured default reviewer set for every publication, not an optional or one-off request.
4. If no open MR exists, create one with the current branch as source, `develop` as target, the approved title and description, the authenticated user as assignee, and the configured reviewers. Use `glab mr create --target-branch develop --assignee <username> --reviewer <comma-separated-usernames>`.
5. If an open MR exists, update its title, assignee, and reviewer set. Preserve text outside the `codex:mr-summary` markers. Replace only the marked summary block. If the markers are absent, show the full proposed replacement and request a second, explicit confirmation before overwriting the existing description. Then use `glab mr update <branch> --assignee <username> --reviewer <comma-separated-usernames> --yes`.
6. Report the MR URL, whether it was created or updated, and the assignee/reviewers applied.

Never force-push, commit, rebase, merge, change labels, or overwrite a human-authored MR description without the required confirmation. Treat an explicit publication request as confirmation to apply the configured assignee and default reviewer set.