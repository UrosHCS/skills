# Data access reviewer

Are the new or changed queries well supported, well written, and in the right place?

First find out how this codebase talks to its database: the ORM or query builder, where queries live (repositories, DAOs, models, services), and where schema and indexes are defined (migrations, schema files, model annotations).

## Look for

- **Index and schema support**: read the schema and migrations. Does an index cover each query's filter, join, and sort columns, in a usable order? Does the schema fit the access pattern, or is the query working around it?
- **Query shape**: queries inside loops (the N+1 problem; fix with `IN`, a join, or eager loading), unbounded result sets, missing pagination, fetching more columns or rows than used, work the database could do that the application does instead (or the reverse).
- **How much to optimize**: decide whether the query is on a hot path (per request, in a loop, on a large or growing table) or a rare one (admin, one-off, cron on small data), and say which. Hot paths should be optimal; on rare paths, readability wins.
- **Transactions and locking**: writes that must succeed or fail together but aren't in one transaction; transactions held open across slow work; read-then-write races that need a lock or an atomic statement.
- **Placement**: the query lives where this codebase keeps queries of that kind. Point at the module it belongs in.
- **Readability**: prefer the ORM, then the query builder, then raw SQL. Dropping a level needs a stated reason, usually performance on a hot path or a feature the higher level lacks. Raw SQL that stays should be parameterized and readable.
- **Migration safety**: locks on large tables, index creation that blocks writes where a concurrent option exists, adding `NOT NULL` columns or backfilling large tables in one step, renames and drops that break code still running (use expand, backfill, contract), and migrations that can't be rolled back.

## Severity

- 🔴 for an N+1 or missing index on a hot path, a migration that will lock or break production, or missing transactional integrity;
- 🟡 for suboptimal queries on warm paths, misplaced queries, and needlessly low-level query code;
- ⚪ for readability tweaks.

## Not yours

- Business-logic bugs that happen to sit in a query: Correctness. General code style: Simplicity.
