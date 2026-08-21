+++
title = "PostgreSQL VACUUM and Autovacuum, Explained"

[taxonomies]
tags = ["PostgreSQL", "Databases"]
+++

If you've run PostgreSQL in production long enough, you've eventually
looked at `pg_stat_user_tables`, seen a `dead_tup` count in the millions,
and wondered what's going on. The answer is almost always: not enough
vacuuming.

<!-- more -->

## Why Postgres needs vacuuming at all

Postgres uses a technique called MVCC (Multi-Version Concurrency Control)
to let multiple transactions read and write the same table without
blocking each other. Instead of updating a row in place, an `UPDATE`
creates a brand new version of the row and marks the old one as no longer
current. A `DELETE` doesn't actually remove the row either — it just marks
it as dead.

This is great for concurrency: readers never block writers, and writers
never block readers. But it means old row versions pile up on disk. Those
are called **dead tuples**, and something has to go clean them up.
That something is `VACUUM`.

## What VACUUM actually does

Running `VACUUM` on a table:

- Scans the table for dead tuples and marks the space they occupied as
  reusable by future inserts and updates (plain `VACUUM` does **not**
  return that space to the operating system — it just makes it available
  for Postgres to reuse within the table).
- Updates the visibility map, which lets Postgres skip pages that contain
  only live, all-visible rows during future scans — a meaningful speedup
  for both vacuum itself and index-only scans.
- Updates planner statistics if you run `VACUUM ANALYZE`.
- Prevents transaction ID wraparound, a much scarier problem where
  Postgres's internal transaction counter could theoretically wrap around
  and make old rows look like they're from the future. Vacuum freezes old
  rows to prevent this from ever becoming a real risk.

If you actually want the disk space back, you need `VACUUM FULL`, which
rewrites the entire table into a new file — but it takes an exclusive lock
on the table for the duration, so it's not something to run casually on a
busy production table.

## Autovacuum: the part you shouldn't turn off

Postgres ships with a background process, autovacuum, that watches your
tables and triggers a vacuum automatically once a table has enough dead
tuples. The default thresholds are conservative:

```
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
```

In practice this means: vacuum a table once roughly 20% of it, plus 50
rows, are dead. That default is tuned for small-to-medium tables. On a
large, high-churn table — think a queue table or a table with a hot
`updated_at` column — 20% of a 50-million-row table is 10 million dead
rows before autovacuum even fires, which is often already too much.

Common tuning moves for busy tables:

- Lower `autovacuum_vacuum_scale_factor` (or set a table-specific override
  with `ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.05)`) so
  vacuum triggers sooner.
- Increase `autovacuum_vacuum_cost_limit` / decrease
  `autovacuum_vacuum_cost_delay` so each vacuum run does more work per
  cycle, instead of throttling itself into uselessness on a busy table.
- Bump `autovacuum_max_workers` if you have many tables that all need
  attention at once and they're competing for the same worker slots.

## A quick way to check if you're behind

```sql
select relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / greatest(n_live_tup, 1), 4) as dead_ratio,
       last_autovacuum
from pg_stat_user_tables
order by n_dead_tup desc
limit 20;
```

If you see tables with a high dead ratio and a `last_autovacuum` that's
suspiciously old (or null), that table's autovacuum settings probably need
tuning — or something is actively holding a long-running transaction open
and preventing vacuum from cleaning up rows that are technically dead but
still "visible" to that old transaction.

The single most common real-world cause of vacuum falling behind isn't bad
settings — it's a long-idle transaction (an app connection left open in a
transaction, or a forgotten `BEGIN` in a psql session) that pins the
oldest-needed row version in place indefinitely. Checking
`pg_stat_activity` for old transactions is usually step one before touching
any autovacuum knob.
