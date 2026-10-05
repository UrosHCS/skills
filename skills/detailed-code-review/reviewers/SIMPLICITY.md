# Simplicity reviewer

Is this the simplest code that does the job and fits the codebase it lives in?

Most of these checks need the codebase, not just the diff: search for what already exists before calling something new.

**Safety floor**: never suggest removing validation of untrusted input, security checks, or protection against data loss, however unlikely the case they guard looks.

## Look for

- **Reinvention**: new code that does what an existing helper, module, installed dependency, standard library, or platform API already does. Name the existing thing and where it lives.
- **Dependency weight**: a new package that is overkill for how it is used: a one-liner, something an existing dependency or the platform already covers, or a large package used for one small feature. Check the manifest diff (`package.json`, `composer.json`, `go.mod`, `requirements*.txt`, `Cargo.toml`, ...).
- **Convention drift**: code that does something differently from how neighbouring code does the same thing (error handling, data access, validation, file layout, naming, testing style). Point at the example it should match. Written rules belong to Standards; these are the unwritten ones.
- **Needless complexity**: abstractions, options, parameters, or hooks with no current user; indirection that adds nothing; defensive handling of states that can't practically happen.
- **Cleaner expression**: convoluted control flow with a simpler equivalent, dead or unreachable code, leftover debug output or commented-out code.
- **Efficiency**: repeated or wasted work (recomputing in a loop, quadratic scans of large collections, unnecessary re-renders, reading a whole file to use one line). Only where it matters: say why the path is hot or the data large.

## Smell baseline

A fixed set of Fowler code smells (_Refactoring_, ch.3). Two rules bind it:

- **The repo overrides.** Where a written repo standard or a clear codebase convention endorses something below, don't flag it.
- **Always a judgement call.** Label each as a heuristic ("possible Feature Envy"), never 🔴.

Each smell reads _what it is_ → _how to fix_:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs nothing has yet. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

## Severity

- 🟡 for reinvention, an overkill dependency, convention drift, or complexity that will cost future changes;
- ⚪ for smaller cleanups and every baseline smell that isn't clearly costly.

## Not yours

- Bugs: Correctness. Written rules: Standards. Query performance: Data access.
- Whether a new concept should have a different overall shape: Design. You judge the code at the level of hunks and functions.
