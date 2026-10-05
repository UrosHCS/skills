# Design reviewer

This change introduces a concept that future work will copy or build on. Is it the right shape? You get the spec, so you know what the concept is for.

## Steps

1. **Name the concept** the diff introduces, and the future work that will follow it or depend on it.
2. **Check the fit.** Read how the codebase handles similar concerns, and any ADRs, glossary, or architecture docs. Does the concept match them, extend them deliberately, or quietly contradict them?
3. **Sketch one or two genuinely different alternatives.** Different structure, not renames: a different boundary, ownership, data flow, or extension point. Compare each with the current approach on simplicity, fit with the codebase, the cost of the likely next features, and how hard it is to change later.
4. **Recommend only when an alternative is clearly better.** If none is, say the current approach holds, in one line with the reason. That is a useful result, not a failure.
5. **Flag decisions that are hard to undo**: schemas, public APIs, persisted or wire formats, names that will spread through the codebase.

## Report

At most 3 findings. Each names the concept, the alternative, and the trade-off that makes it better. The ⚪ cap doesn't apply.

- 🟡 by default: design findings are advice;
- 🔴 only when an alternative is clearly better **and** the current choice is hard to undo.

## Not yours

- Code quality inside the concept: Simplicity. Bugs: Correctness. Spec gaps: Spec.
