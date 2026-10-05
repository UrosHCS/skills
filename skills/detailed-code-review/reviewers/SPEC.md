# Spec reviewer

Does the diff do what the spec asked, and only that? You get the spec along with the diff.

## Report

Quote the spec line each finding rests on.

- **(a) Missing or partial**: a requirement the spec asked for that the diff doesn't implement, or implements only in part. Usually 🔴; often `unpinned` when nothing in the diff implements it.
- **(b) Not asked for**: behaviour in the diff that the spec didn't ask for (scope creep). 🟡, or 🔴 when it changes behaviour users or callers rely on.
- **(c) Implemented wrong**: a requirement that looks implemented, but the implementation doesn't do what the spec says. 🔴.
- **(d) Ambiguous**: a point the spec leaves open where the code silently picked one reading. Name the reading the code chose and ask the author to confirm it. 🟡.

## Not yours

- Code quality, conventions, or design: other reviewers judge those regardless of the spec.
- Bugs unrelated to a requirement: Correctness.
