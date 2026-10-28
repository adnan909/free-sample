---
name: graphql-n-plus-one
description: Use when writing or reviewing GraphQL resolvers in a TypeScript + Postgres backend — finds N+1 query patterns and fixes them with per-request DataLoaders. Triggers on new resolvers for list or nested fields, slow GraphQL queries, or "too many queries" in logs.
---

# GraphQL N+1 detection and DataLoader fixes

An N+1 happens when a list resolver returns N parents and a field resolver on each parent runs its own query. One request becomes 1 + N round trips to Postgres. This skill finds those and fixes them without changing the schema.

## Step 1: Find the N+1

Check every resolver for a **field on a type that can appear in a list** (`Post.author`, `Order.items`, `Project.owner`). If that resolver calls the database directly with the parent's id, it is an N+1.

```ts
// N+1: runs once per post
Post: {
  author: (post, _args, ctx) =>
    ctx.db.query('SELECT * FROM users WHERE id = $1', [post.authorId]),
}
```

Confirm before fixing: enable query logging (or count queries in a test, see Step 4) and run a query that fetches 20+ parents. If query count grows with the list size, it is confirmed.

## Step 2: Create loaders per request, never globally

DataLoader caches by key. A module-level loader leaks data between users and requests. Always build loaders in the context factory.

```ts
// src/loaders.ts
import DataLoader from 'dataloader';
import type { Pool } from 'pg';

export function createLoaders(db: Pool) {
  return {
    userById: new DataLoader<string, User | null>(async (ids) => {
      const { rows } = await db.query<User>(
        'SELECT * FROM users WHERE id = ANY($1::uuid[])',
        [ids],
      );
      const byId = new Map(rows.map((r) => [r.id, r]));
      return ids.map((id) => byId.get(id) ?? null); // same order and length as ids
    }),
  };
}

// context factory
context: () => ({ db, loaders: createLoaders(db) })
```

Rules for the batch function:

- Return an array with **exactly the same length and order** as `ids`. Missing rows become `null`, never skipped.
- Use `= ANY($1::<type>[])` with one array parameter, not a string-built `IN (...)` list.
- For one-to-many (`Order.items`), group rows by foreign key and return an array per key (`[]` when none).

```ts
itemsByOrderId: new DataLoader<string, OrderItem[]>(async (orderIds) => {
  const { rows } = await db.query<OrderItem>(
    'SELECT * FROM order_items WHERE order_id = ANY($1::uuid[]) ORDER BY created_at',
    [orderIds],
  );
  const groups = new Map<string, OrderItem[]>();
  for (const row of rows) {
    const list = groups.get(row.orderId) ?? [];
    list.push(row);
    groups.set(row.orderId, list);
  }
  return orderIds.map((id) => groups.get(id) ?? []);
}),
```

## Step 3: Use the loader in the resolver

```ts
Post: {
  author: (post, _args, ctx) => ctx.loaders.userById.load(post.authorId),
}
```

After a mutation changes a row, call `ctx.loaders.userById.clear(id)` before returning it, so the response doesn't show cached stale data.

## Step 4: Prove it with a test

Add a test that counts queries, so a future change can't silently reintroduce the N+1.

```ts
it('loads authors for 25 posts in a constant number of queries', async () => {
  const spy = jest.spyOn(pool, 'query');
  await execute(`{ posts(first: 25) { id author { id name } } }`);
  expect(spy.mock.calls.length).toBeLessThanOrEqual(2); // posts + one batched users query
});
```

## Don't

- Don't batch with `Promise.all` over individual queries; that is still N queries.
- Don't put authorization inside the batch function. Check permissions in the resolver, after loading, so the cache never mixes users' access.
- Don't add a loader for a field that is never resolved inside a list; it adds complexity for no gain.

## Done when

- Query count for a list of 25 parents is constant (1 to 3), not 26+.
- Every loader is created in the per-request context.
- A query-count test covers each fixed field.
