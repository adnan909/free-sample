---
name: postgres-safe-migrations
description: Use when writing or reviewing a Postgres schema migration for a live app — adding columns, indexes, constraints, renames or drops without locking tables or breaking running code. Triggers on any new migration file, schema change, or "add a column/index" request.
---

# Safe Postgres migrations for a live app

A migration is safe when it never holds a lock that blocks normal reads and writes for more than a moment, and when the old and new versions of the app can both run against the schema while a deploy rolls out.

## Step 1: Classify every statement

Before writing the migration, sort each change into one row below and follow its recipe.

| Change | Risk | Safe recipe |
| --- | --- | --- |
| Add nullable column | Low | Plain `ADD COLUMN` |
| Add column with a constant default | Low on Postgres 11+ | Plain `ADD COLUMN ... DEFAULT x` |
| Add NOT NULL column | High | Add nullable → backfill in batches → add `CHECK ... NOT VALID` → `VALIDATE` → `SET NOT NULL` |
| Add index | High (blocks writes) | `CREATE INDEX CONCURRENTLY`, in its own migration with no transaction |
| Add foreign key | High | `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` in a later step |
| Rename column or table | Breaks running code | Expand and contract (Step 3) |
| Drop column | Breaks running code | Remove from code first, deploy, then drop |
| Change column type | High (rewrites table) | New column + backfill + switch reads, never `ALTER TYPE` on a big table |

## Step 2: Always set a lock timeout

Every migration starts with a short lock timeout, so it fails fast instead of queueing behind a long query and blocking every request behind it.

```sql
SET lock_timeout = '3s';
SET statement_timeout = '60s';
```

If it fails on the timeout, retry later. Never raise the timeout to "make it work".

## Step 3: Renames use expand and contract

Never rename in one step. Running app instances still use the old name during deploy.

1. **Expand:** add the new column; write to both columns in code.
2. **Backfill:** copy old to new in batches (Step 4).
3. **Switch reads** to the new column; deploy.
4. **Contract:** stop writing the old column; deploy; then drop it in a later migration.

## Step 4: Backfill in batches

One giant `UPDATE` locks rows for its whole run and bloats the table. Batch by primary key:

```sql
UPDATE projects
SET status_v2 = status
WHERE id IN (
  SELECT id FROM projects
  WHERE status_v2 IS NULL
  ORDER BY id
  LIMIT 5000
);
-- repeat until 0 rows updated; run from a script, not inside the migration
```

## Step 5: Index example

```sql
-- migration file must disable its transaction (most tools: a flag or comment)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_issues_project_created
  ON issues (project_id, created_at DESC);
```

If a concurrent build fails, it leaves an `INVALID` index. Drop it with `DROP INDEX CONCURRENTLY` and retry.

## Review checklist

- [ ] `lock_timeout` set at the top
- [ ] No `CREATE INDEX` without `CONCURRENTLY` on an existing table
- [ ] No new `NOT NULL` or foreign key validated in the same step it's added
- [ ] No rename, drop or type change that running code still depends on
- [ ] Backfills are batched and run outside the migration
- [ ] A down migration exists, or the PR says why it can't

## Done when

Every statement maps to a row in Step 1, follows its recipe, and the checklist passes.
