# Topic 7 — `sort()`

Control the **order** of returned documents.

```ts
// ascending (small → big)
const asc = await Product.find().sort({ price: 1 });

// descending (big → small)
const desc = await Product.find().sort({ price: -1 });
```

Result for `{ price: 1 }`:

```text
100 → 200 → 300 → 500 → 800
```

Result for `{ price: -1 }`:

```text
800 → 500 → 300 → 200 → 100
```

## Multiple fields (tie-breaking)

```ts
await Product.find().sort({ price: -1, name: 1 });
// 1. sort by price descending
// 2. when prices tie, sort by name ascending
```

## Equivalent forms

```ts
// object
Product.find().sort({ price: 1 });

// string
Product.find().sort("price name");

// array of tuples
Product.find().sort([["price", 1], ["name", 1]]);
```

## Real API example

```text
GET /products?sort=price&order=desc
```

```ts
const sortField = String(req.query.sort ?? "name");
const direction = req.query.order === "desc" ? -1 : 1;

const products = await Product.find().sort({ [sortField]: direction });
```

## Performance note

Sorting on an **unindexed** field forces an in-memory sort (limited to 100MB) and gets slower as data grows. Add an index for the sort patterns you actually ship:

```ts
ProductSchema.index({ price: 1 });
ProductSchema.index({ price: -1, name: 1 });
```

Verify with:

```ts
await Product.find().sort({ price: -1 }).explain("executionStats");
```