---
name: detailed-code-review
description: Detailed code review, where parallel specialist reviewers check one diff and return line-pinned findings. Use for a detailed review of a branch, a PR/MR, unstaged changes, or everything since a Git ref.
---

# Detailed Code Review

Reviews one diff through up to six **lenses**, each run as a parallel sub-agent in **isolation**: a reviewer sees the diff, its brief, and (for two lenses) the spec, never another reviewer's conclusions. A change can pass one lens and fail another: it follows every written rule but builds the wrong thing, or does exactly what was asked but rebuilds a helper the codebase already has. Separate reports stop one lens from masking another, so aggregation preserves the isolation too.

| Reviewer | Runs | Gets the spec | Brief |
|---|---|---|---|
| **Standards**: does the code follow the repo's written rules? | always | no | [reviewers/STANDARDS.md](reviewers/STANDARDS.md) |
| **Spec**: does the code do what the spec asked, and only that? | when a spec exists | yes | [reviewers/SPEC.md](reviewers/SPEC.md) |
| **Simplicity**: is this the simplest code that fits the codebase? | always | no | [reviewers/SIMPLICITY.md](reviewers/SIMPLICITY.md) |
| **Correctness**: does the code work, safely, for real inputs? | always | no | [reviewers/CORRECTNESS.md](reviewers/CORRECTNESS.md) |
| **Design**: is a new precedent the right shape for future work? | when triggered | yes | [reviewers/DESIGN.md](reviewers/DESIGN.md) |
| **Data access**: are the queries well supported, written, and placed? | when triggered | no | [reviewers/DATA.md](reviewers/DATA.md) |

Telling a reviewer what a change is meant to do measurably makes it miss problems, so only the two lenses that judge intent get the spec. Rules every reviewer follows (read-only, line pinning, finding format, severity, confidence, and which reviewer owns which concern) live in [reviewers/COMMON.md](reviewers/COMMON.md).

## Inputs

- **Diff**: one scope, which fixes the diff command, the commit list, and any untracked files:

  | Scope | Diff command | Commit list | Untracked files |
  |---|---|---|---|
  | Fixed point (commit SHA, branch, tag, `main`, `HEAD~5`, ...) | `git diff <fixed-point>...HEAD` | `git log <fixed-point>..HEAD --oneline` | none |
  | Unstaged changes | `git diff` | none | `git ls-files --others --exclude-standard`, read in full by reviewers |
  | Supplied by a calling skill | as given | as given; for an `A...B` range with none given, `git log A..B --oneline` | none |

  The three-dot range diffs against the merge-base. A PR/MR the user names is a fixed point at its target branch, reviewed with its head checked out as `HEAD`.
- **Spec**: a path, the spec text, or `none`.
- **Review root** (optional, from a calling skill): the absolute path of the checkout the diff lives in, such as a worktree. Reviewers run commands and read files under it; pins keep the diff's own paths.
- **Overrides** (optional): `+design` / `-design` and `+data` / `-data` force an optional reviewer on or off.

**Called by another skill**: the caller's inputs are final, so run without asking the user anything. On a missing or invalid input, stop with a clear error. Return the report; posting or acting on findings is the caller's job.

## Process

### 1. Pin the diff

When running interactively with no scope given, ask for a fixed point or unstaged changes.

Done when the diff command runs (a fixed point also resolves with `git rev-parse <fixed-point>`) and the scope is non-empty: for unstaged changes, `git diff` or the untracked list has content; otherwise the diff does. On failure, stop with the error before spawning anyone. Use this exact diff command everywhere below.

### 2. Pin the spec

Use the spec the user or caller supplied. When running interactively with none supplied, ask where it is. With no spec, skip the Spec reviewer and record why. The spec comes only from the user or caller, never from commit messages.

### 3. Decide the optional reviewers

Read the diff command's `--stat` output, keeping its revision range and path filters (for example `git diff --stat <base_sha>...HEAD`), and skim the diff itself. For unstaged reviews, also skim the untracked files: `--stat` omits them. Overrides win over everything below.

**Design** runs when the change sets a **precedent**, a concept future work will copy or build on:

- a new module, package, service, or layer boundary;
- a new abstraction others will implement or extend (interface, base class, registry, plugin or event mechanism, middleware);
- a new public API, contract, data model, or persisted format;
- a new cross-cutting mechanism (caching, auth, error handling, state management, background jobs, configuration);
- a new pattern later features are expected to follow.

Size alone never triggers it: a large feature that follows existing patterns sets no precedent.

**Data access** runs when the change adds or edits at least one non-trivial query or schema change: migrations, schema or index definitions, ORM models and query calls, repository or DAO code, query builder chains, raw SQL strings. Trivial means a single-row lookup by primary key, or an edit that leaves what a query does unchanged (rename, formatting).

Done when Design and Data access each have a run-or-skip decision with a one-line reason.

### 4. Spawn the reviewers in parallel

Spawn every reviewer that runs in one message, so they run concurrently. Each prompt contains exactly:

- the absolute paths of `reviewers/COMMON.md` and that reviewer's brief, to read before anything else (sub-agents can't resolve paths relative to this skill);
- the diff command, the commit list, and, for unstaged changes, the untracked files;
- the review root, when one was given;
- for the reviewers that get the spec, the spec path or text.

That list is the whole prompt. Isolation means no summary of what the change is for and no other reviewer's findings.

### 5. Aggregate

1. **Drop low-confidence findings.** Count them per reviewer.
2. **Dedupe across reviewers.** When two reviewers flag the same lines for the same underlying problem, keep the finding in the section of the reviewer that owns that concern (the ownership table in `COMMON.md`) and append `(also flagged by <Reviewer>)`.
3. **Cap ⚪ findings** at the 5 most useful per reviewer; count the rest.
4. **Render** the report below. Every finding not deduped stays in its reviewer's section, in that reviewer's order.

Done when every finding a reviewer returned is either in the report or in a `Dropped` count.

## Report

```markdown
# Detailed code review: <diff description>

Reviewers: Standards ✓ · Spec ✓ · Simplicity ✓ · Correctness ✓ · Design: skipped, <reason> · Data access ✓ (<reason>)

## Standards

- 🔴 `path/to/file.ts` · line 42 · high: <finding>
  - Evidence: <quoted code, rule, or spec line>
  - Fix: <suggested change>

## Spec

No findings.

...

## Summary

- Standards: 2 findings (🔴 0 · 🟡 2 · ⚪ 0). Worst: <one line>
- ...
- Dropped: 3 low-confidence (Simplicity 2, Correctness 1); 2 ⚪ over the cap (Simplicity).
```

- One section per reviewer that ran, in the table's order; the section a finding sits in names its reviewer. A skipped reviewer gets no section, only its reason in the `Reviewers:` line.
- Reviewer output that isn't a finding (`No findings.`, `No documented standards found.`, Design's one-line verdict that the current approach holds) goes into that reviewer's section as written. A reviewer that errored gets a section saying so.
- The summary names the worst issue **within each reviewer**. A single winner across reviewers would rerank what isolation keeps apart.
