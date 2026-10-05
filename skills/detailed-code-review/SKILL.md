---
name: detailed-code-review
description: Review branch, PR/MR, or unstaged changes with independent specialist reviewers and evidence-backed, line-pinned findings. Use for detailed code reviews or reviews since a specified Git ref.
---

# Detailed Code Review

Reviews one diff along up to six separate axes. Each axis runs as its own **parallel sub-agent** so no reviewer's context or conclusions leak into another's, then this skill aggregates the reports without merging them.

| Reviewer | Runs | Gets the spec | Brief |
|---|---|---|---|
| **Standards**: does the code follow the repo's written rules? | always | no | [reviewers/STANDARDS.md](reviewers/STANDARDS.md) |
| **Spec**: does the code do what the spec asked, and only that? | when a spec exists | yes | [reviewers/SPEC.md](reviewers/SPEC.md) |
| **Simplicity**: is this the simplest code that fits the codebase? | always | no | [reviewers/SIMPLICITY.md](reviewers/SIMPLICITY.md) |
| **Correctness**: does the code work, safely, for real inputs? | always | no | [reviewers/CORRECTNESS.md](reviewers/CORRECTNESS.md) |
| **Design**: is a new concept the right shape for future work? | when triggered | yes | [reviewers/DESIGN.md](reviewers/DESIGN.md) |
| **Data access**: are the queries well supported, written, and placed? | when triggered | no | [reviewers/DATA.md](reviewers/DATA.md) |

Rules every reviewer follows (read-only, line pinning, finding format, severity, confidence) live in [reviewers/COMMON.md](reviewers/COMMON.md).

Only Spec and Design see the spec. Telling a reviewer what a change is meant to do measurably makes it miss problems, so the others judge the code on its own.

## Inputs

- **Diff**: one of
  - a fixed point (commit SHA, branch, tag, `main`, `HEAD~5`, ...): review `git diff <fixed-point>...HEAD` (three-dot, so against the merge-base), commits from `git log <fixed-point>..HEAD --oneline`;
  - unstaged changes: review `git diff`, plus any untracked files from `git status --porcelain`, which reviewers read in full;
  - a diff command supplied by a calling skill (for example `git diff <base_sha>...HEAD`), optionally with its commit list command. If none is given and the command is a `A...B` range, derive the commit list as `git log A..B --oneline`.
- **Spec**: a path, the spec text, or "none".
- **Overrides** (optional): `+design` / `-design` and `+data` / `-data` force an optional reviewer on or off.

**Called by another skill**: when the caller supplies the diff and the spec, never ask the user anything. If an input is missing or invalid, stop with a clear error instead of prompting. Return the report; posting or acting on findings is the caller's job.

## Process

### 1. Pin the diff

If the user gave no fixed point and didn't ask for unstaged changes, ask which.

Capture the selected diff command once and use it throughout the review. If a fixed point was supplied, confirm it resolves (`git rev-parse <fixed-point>`); unstaged reviews need no fixed point. Stop here if the selected diff command fails.

For unstaged reviews, collect untracked files with `git ls-files --others --exclude-standard` and stop only when both `git diff` and that file list are empty. For other review scopes, stop when the selected diff is empty. Resolve invalid refs and empty review scopes before spawning reviewers.

### 2. Pin the spec

Use the spec the user or caller supplied. If none was supplied and you're running interactively, ask where it is. If there isn't one, skip the Spec reviewer and say so in the report; never fabricate a spec from commit messages.

### 3. Decide the optional reviewers

Read the selected diff command's `--stat` output, preserving its revision range and path filters (for example, `git diff --stat <base_sha>...HEAD`), and skim that same diff. For unstaged reviews, also inspect the collected untracked files when selecting reviewers; they do not appear in `git diff --stat`. Overrides win over everything below. Record each decision with a one-line reason for the report.

**Design** runs when the change introduces a concept future work will copy or build on:

- a new module, package, service, or layer boundary;
- a new abstraction others will implement or extend (interface, base class, registry, plugin or event mechanism, middleware);
- a new public API, contract, data model, or persisted format;
- a new cross-cutting mechanism (caching, auth, error handling, state management, background jobs, configuration);
- a new pattern later features are expected to follow.

It does not run for features that follow existing patterns, however large.

**Data access** runs when the change adds or edits at least one non-trivial query or schema change: migrations, schema or index definitions, ORM models and query calls, repository or DAO code, query builder chains, raw SQL strings. Trivial means a single-row lookup by primary key, or an edit that doesn't change what a query does (rename, formatting).

### 4. Spawn the reviewers in parallel

Spawn every reviewer that runs in one message, so they run concurrently. Pass reviewers the **absolute paths** of this skill's files; sub-agents can't resolve paths relative to this skill.

Every reviewer prompt includes:

- the absolute paths of `reviewers/COMMON.md` and that reviewer's brief, with the instruction to read both before anything else;
- the diff command (and, for unstaged changes, the untracked files) and the commit list;
- Spec and Design only: the spec path or text.

Add nothing else: no summary of what the change is for, and no findings from other reviewers.

### 5. Aggregate

1. **Drop low-confidence findings.** Count them per reviewer.
2. **Dedupe across reviewers.** When two reviewers flag the same lines for the same underlying problem, keep the finding in the section of the reviewer whose brief owns that concern and append `(also flagged by <Reviewer>)`. Don't otherwise move, merge, or rerank findings.
3. **Cap Consider findings** at the 5 most useful per reviewer; count the rest.
4. **Render** the report below.

## Report

```markdown
# Detailed code review: <diff description>

Reviewers: Standards ✓ · Spec ✓ · Simplicity ✓ · Correctness ✓ · Design: skipped, <reason> · Data access ✓

## Standards

- 🔴 `path/to/file.ts` · line 42 · high: <finding>
  - Evidence: <quoted code, rule, or spec line>
  - Fix: <suggested change>

No findings. ← when a reviewer found nothing

## Spec
...

## Summary

- Standards: 2 findings (🔴 0 · 🟡 2 · ⚪ 0). Worst: <one line>
- ...
- Dropped: 3 low-confidence (Simplicity 2, Correctness 1); 2 ⚪ over the cap (Simplicity).
```

One section per reviewer that ran, in the table's order. A skipped reviewer gets no section; its reason is in the `Reviewers:` line. The reviewer is the section a finding sits in.

The summary names the worst issue **within each reviewer**, never one winner across reviewers. That is the reranking the separation exists to prevent.

## Why separate reviewers

A change can pass one axis and fail another:

- code that follows every written rule but implements the wrong thing passes Standards and fails Spec;
- code that does exactly what was asked but rebuilds a helper the codebase already has passes Spec and fails Simplicity;
- clean, well-placed code with an off-by-one error passes Simplicity and fails Correctness.

Separate reports stop one axis from masking another, and narrow briefs keep each reviewer's attention where its lens is sharpest.
