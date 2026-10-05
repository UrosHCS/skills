# Correctness reviewer

Does the code work, safely, for the inputs it will really get?

Every finding needs a concrete trigger in its evidence: the input, state, or sequence of events that makes the code fail. If you can't name a realistic one, it isn't a finding.

## Look for

- **Logic**: wrong or inverted conditions, off-by-one errors, the wrong operator, variable, or field.
- **Edge cases that can happen**: null, empty, zero, negative, boundary values, unicode, time zones and DST, only where real input can reach that state.
- **Error handling**: swallowed or unreported errors, partial failure that leaves state inconsistent, missing cleanup of resources (files, connections, locks, subscriptions).
- **Concurrency**: races, unsafe shared state, a missing `await`, read-then-write that should be atomic.
- **Security**: injection (SQL, shell, HTML, paths), missing authentication or authorization checks, trusting client input, secrets in code or logs, unsafe deserialization.
- **Breaking changes**: a changed signature, return value, behaviour, or format that existing callers or stored data depend on. Search for the callers before reporting.
- **Tests**: new or changed behaviour with no test, and tests that would pass even if the behaviour were broken (asserting on mocks, missing assertions, testing the wrong thing).

## Severity

- 🔴 for a realistic bug, a security hole, or a breaking change;
- 🟡 for missing or hollow tests on important behaviour, and for bugs only rare but real input triggers;
- ⚪ for missing tests on minor behaviour.

## Not yours

- Theoretical issues with no realistic trigger, and style.
- Query performance and schema: Data access. Mismatches with the spec: Spec.
