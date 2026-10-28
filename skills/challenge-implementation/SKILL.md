---
name: challenge-implementation
description: "Challenge a finished diff for a materially smaller approach to the same task: an existing mechanism, a platform feature, or parts that needn't exist. Use after implementing and before code review, or when asked whether a change is overengineered."
---

# Challenge Implementation

One **challenger** sub-agent asks a single question of a finished change: is there a materially smaller way to do the whole task? It works at the level of the approach (components, layers, dependencies, state), not lines.

The challenger runs in **isolation**: it sees the spec, the diff, and the codebase, never the implementer's reasoning. You orchestrate and relay; read only this file, never the challenger's brief, so its instructions stay out of your context.

## Inputs

- **Diff**: one scope. A fixed point (commit SHA, branch, `main`, ...) is reviewed with `git diff <fixed-point>...HEAD`. Unstaged changes are reviewed with `git diff` plus the untracked files from `git ls-files --others --exclude-standard`. A calling skill may supply the diff command directly.
- **Spec**: a path or the spec text. The challenger needs it to know what "the same task" is.
- **Review root** (optional, from a calling skill): the absolute path of the checkout the diff lives in.

**Called by another skill**: the caller's inputs are final, so run without asking the user anything. On a missing or invalid input, stop with a clear error. When running interactively with an input missing, ask for it.

## Process

1. **Pin the diff.** Done when the diff command runs and the scope is non-empty: the diff has content, or for unstaged changes, `git diff` or the untracked list does. On failure, stop with the error.
2. **Spawn the challenger.** One sub-agent whose prompt contains exactly:
   - the absolute path of [CHALLENGER.md](CHALLENGER.md), to read before anything else;
   - the diff command and, for unstaged changes, the untracked files;
   - the spec path or text;
   - the review root, when one was given.

   That list is the whole prompt: no summary of the change and no account of why it was built this way.
3. **Return the challenger's report verbatim.** Acting on it is the caller's job.
