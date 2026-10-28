---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user.

Use /write-tests where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done, use /challenge-implementation on the work. Adopt each alternative unless it gives up something the spec requires or you can name a concrete cost the challenger missed; after adopting any, rerun typechecking and the full test suite. In your final message, list each alternative as adopted or rejected with the reason, and carry over its questions for the user.

Then use /detailed-code-review to review the work.

When the review is finished, use /apply-valid-feedback to handle the reviewer's points.

Do not commit your work.
