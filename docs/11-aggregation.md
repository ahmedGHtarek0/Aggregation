# Topic 11 — Aggregation Pipeline

Aggregation turns a collection into a **pipeline**: each stage receives the previous stage's documents, transforms them, and passes them on.

```text
Products → $match → $group → $sort → Final Result
```

Great for: filtering, grouping, joining, calculations, reshaping, and computing totals/averages — the stuff plain `find()` can't do.

## Core stages

```ts
const pipeline = [
  // 1. Filter FIRST (smallest data set into later stages)
  { $match: { category: "Food" } },

  // 2. Group & compute
  {
    $group: {
      _id: "$category",
      avgPrice: { $avg: "$price" },
      totalProducts: { $sum: 1 },
      revenue: { $sum: { $multiply: ["$price", "$quantity"] } }
    }
  },

  // 3. Sort the grouped result
  { $sort: { avgPrice: -1 } },

  // 4. Reshape the output
  { $project: { _id: 0, category: "$_id", avgPrice: 1 } }
];

const stats = await Product.aggregate(pipeline);
```

### `$group` — the heart of aggregation

```ts
{
  $group: {
    _id: "$category",                 // grouping key
    count: { $sum: 1 },               // number of docs
    avgPrice: { $avg: "$price" },
    minPrice: { $min: "$price" },
    maxPrice: { $max: "$price" },
    names: { $addToSet: "$name" },    // unique values
    total: { $sum: "$price" }
  }
}
```

### `$lookup` — the real JOIN

```ts
const productsWithOwners = await Product.aggregate([
  {
    $lookup: {
      from: "users",          // target raw collection name (plural)
      localField: "owner",
      foreignField: "_id",
      as: "owner"             // always an array
    }
  },
  { $unwind: "$owner" },                     // array → single object
  { $project: { "owner.passwordHash": 0 } }  // hide sensitive fields
]);
```

### `$unwind` — explode arrays

One output doc per element:

```ts
await Product.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", products: { $addToSet: "$name" } } }
]);
```

### `$addFields`, `$count`, `$limit`, `$sample`

```ts
await Product.aggregate([
  { $addFields: { discounted: { $multiply: ["$price", 0.9] } } },
  { $match: { discounted: { $lte: 450 } } },
  { $count: "discountedCount" }            // [{ discountedCount: 2 }]
]);

await Product.aggregate([{ $sample: { size: 3 } }]); // random docs
```

## Pipeline order — performance matters

```text
1  $match  +  $project          shrink data early (reduces work downstream)
2  $lookup +  $unwind           join/widen only when needed
3  $group  +  $addFields        compute
4  $sort   +  $skip/$limit      order + page
5  $project/$count              final shape
```

If `$group`/`$sort` exceed the 100MB memory ceiling, chain `.allowDiskUse(true)` or add `{ $allowDiskUse: true }`.

## `$match` is a plain Mongo query

Everything you learned about filters works inside `$match`:

```ts
{ $match: { category: { $in: ["Food", "Toys"] }, price: { $gte: 200 } } }
```

## find() vs aggregate()

| | `find()` | `aggregate()` |
|---|---|---|
| Idea | "give me documents" | "transform documents" |
| Output | array of raw docs | fully shaped result |
| Group / join / compute | ❌ | ✅ |
| Use `.lean()`? | yes for speed | always plain objects |