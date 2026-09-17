---
name: gitlab-mr-review
description: Review an existing GitLab merge request against the SHAs GitLab stored for that MR, using a two-axis Standards/Spec review and an optional Jira spec, then preview and post inline comments. Use when asked to review a GitLab MR by URL, iid, or the MR for the current branch. Do not use for GitHub pull requests.
disable-model-invocation: true
---

# GitLab MR Review

Two-axis review of the merge request's stored diff (`git diff <base_sha>...HEAD` once HEAD is `head_sha`):

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Both axes run as **parallel sub-agents** so they don't pollute each other's context. Findings are previewed, then posted as inline MR comments only after the user confirms.

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

Do not use `glab mr diff` as the review input. Once HEAD is `head_sha` and the fixed point is `base_sha`, `git diff <base_sha>...HEAD` is the same three-dot diff GitLab stored for this MR version. Confirm `base_sha` resolves and that diff is non-empty before spawning sub-agents.

## Spec source

5. If the MR details contain a jira ticket id in the format `PRO-####`, use `acli jira workitem view PRO-#### --fields "summary,description"` to get the ticket info. Do not follow links from the description or try to view images. Spec's primary source is the jira ticket info (if any); MR title and description are secondary. If both are thin, there is no spec: skip the Spec sub-agent rather than fabricating one.

## Review

Capture the diff command once: `git diff <base_sha>...HEAD`. Also note `git log <base_sha>..HEAD --oneline`.

### Standards sources

Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below: a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads _what it is_ → _how to fix_; match it against the diff:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### Spawn both sub-agents in parallel

Every finding must include the file path as it appears in the diff (new path after renames) and the line it refers to. Use `line` for the new/source side, or `old_line` only for a removed line. Take those numbers from the diff hunk (`@@ -old +new @@` plus the `+`/`-`/context lines), not from reading the file in isolation. If a finding is not about a specific changed line (for example a missing requirement with no implementation site), say that explicitly instead of guessing a line.

**Standards sub-agent prompt** should include:

- The full diff command and commit list.
- The list of standards-source files, **plus the smell baseline pasted in full** (the sub-agent has no other access to it).
- The pin rule above.
- The brief: "Report, per file/hunk where relevant, (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls: documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. For each finding: path, `line` or `old_line` (or explicitly unpinned). Under 400 words."

**Spec sub-agent prompt** should include:

- The diff command and commit list.
- The fetched spec contents, or skip this sub-agent if there is no spec.
- The pin rule above.
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. For each finding: path, `line` or `old_line` (or explicitly unpinned). Under 400 words."

### Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do **not** merge or rerank findings.

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Don't pick a single winner across axes: that's the reranking the separation exists to prevent.

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.

Restore the original branch (or detached SHA) now that the review is done.

## Preview then post

If both axes have zero findings, say so and do not post.

Otherwise show the proposed GitLab comments before posting: path, `line` / `old_line` or unpinned, axis, and the exact `-m` text. Each `-m` is one finding, labelled Standards or Spec. Do not paste the whole review into one message. Post at most 8 inline comments; prefer the most serious findings and put any remainder in one short overview thread. Skip nits and praise. If the MR already has unresolved discussions, mention that in the preview so a second run is an explicit choice. Do not use `--unique` (it cannot combine with `--file`).

Do not post until the user explicitly confirms.

Then post each selected finding as its own inline comment. `--line` is the new/source-side number; `--old-line` is only for a removed line. Use `--line 10:15` for a contiguous range. Omit both line flags for a file-level comment. If a finding has no pin, post it as a general comment on the MR.

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
