# MongoDB & Mongoose — Querying Master Guide

> An advanced, production-ready reference for junior backend developers who want to stop guessing and start querying MongoDB through Mongoose like a professional.

This repository is a **single source of truth** for the most important MongoDB/Mongoose querying concepts: document retrieval, filtering, sorting, pagination, and relationships.

Each major topic lives on its **own git branch** (see [Branches](#-repository-structure--branches)) so you can study, checkout, and experiment with one concept at a time.

---

## Table of Contents

1. [What This Guide Covers](#what-this-guide-covers)
2. [Setup](#setup)
3. [Document Retrieval](#document-retrieval)
    - [`find()`](#1-find)
    - [`findOne()`](#2-findone)
    - [`findById()`](#3-findbyid)
4. [Filtering](#filtering)
    - [$gte, $lte, $gt, $lt](#4-comparison-operators)
    - [`$in`](#5-in)
    - [`$or` and `$and`](#6-or-and--and)
5. [Query Result Control](#query-result-control)
    - [`sort()`](#7-sort)
    - [`limit()`](#8-limit)
    - [`skip()` + Pagination](#9-skip--pagination)
6. [Relationships](#relationships)
    - [`populate()`](#10-populate)
7. [Advanced Topics](#advanced-topics)
8. [Repository Structure & Branches](#-repository-structure--branches)
9. [Cheat Sheet](#cheat-sheet)
10. [Common Pitfalls](#common-pitfalls)

---

## What This Guide Covers

| Category | Topics |
|---|---|
| Retrieval | `find()`, `findOne()`, `findById()` |
| Filtering | `$gte`, `$lte`, `$gt`, `$lt`, `$in`, `$or`, `$and` |
| Result control | `sort()`, `limit()`, `skip()` |
| Pagination | `skip`/`limit` strategy + formulas |
| Relationships | `populate()` (MongoDB's "JOIN") |

---

## Setup

Modern Mongoose is **TypeScript-first**, but every example works identically in JavaScript.

```bash
npm install mongoose
```

```ts
import mongoose from "mongoose";
import { Schema, model, type InferSchemaType, type HydratedDocument } from "mongoose";

const ProductSchema = new Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true, min: 0 },
  category: { type: String, enum: ["Food", "Toys", "Medicine"], index: true },
  owner: { type: Schema.Types.ObjectId, ref: "User" },
  inStock: { type: Boolean, default: true },
  tags: [{ type: String }],
  createdAt: { type: Date, default: Date.now }
});

type Product = InferSchemaType<typeof ProductSchema>;
const Product = model("Product", ProductSchema);
```

Connect once at app startup:

```ts
await mongoose.connect("mongodb://127.0.0.1:27017/your_db");
```

### The Example Dataset

```js
// products
[
  { name: "Dog Food",   price: 500, category: "Food",     owner: ObjectId("123456") },
  { name: "Cat Food",   price: 300, category: "Food",     owner: ObjectId("123457") },
  { name: "Dog Toy",    price: 200, category: "Toys",     owner: ObjectId("123456") },
  { name: "Vitamins",   price: 800, category: "Medicine", owner: ObjectId("123458") },
  { name: "Fish Tank",  price: 1200, category: "Toys",    owner: ObjectId("123457") }
]

// users
[
  { _id: ObjectId("123456"), name: "Ahmed"  },
  { _id: ObjectId("123457"), name: "Sara"   },
  { _id: ObjectId("123458"), name: "Omar"   }
]
```

---

## Document Retrieval

Use the right tool for the job.

### 1. `find()`

Returns **multiple documents** — always an **array**.

```ts
const products = await Product.find();            // all products
const food     = await Product.find({ category: "Food" });
```

```text
find()       → array (possibly empty, but never null)
find()       → [ {Dog Food}, {Cat Food} ]
```

> Even when exactly one document matches, `find()` still resolves to an array. `findOne()` is the correct tool for "give me a single object".

### 2. `findOne()`

Returns **the first matching document** or `null`.

```ts
const dogFood = await Product.findOne({ name: "Dog Food" });
const anyFood = await Product.findOne({ category: "Food" }); // only ONE match, even if 20 exist
```

```text
find()      → []  | [doc, doc, ...]
findOne()   → null | doc
```

### 3. `findById()`

Sugar for `findOne({ _id: id })` — but ensures the query uses the `_id` index.

```ts
const product = await Product.findById("64f123abc...");
// is effectively:
//   await Product.findOne({ _id: "64f123abc..." })
```

REST pattern:

```http
GET /products/:id
```

```ts
app.get("/products/:id", async (req, res) => {
  const product = await Product.findById(req.params.id);
  if (!product) return res.status(404).json({ message: "Not found" });
  res.json(product);
});
```

```text
find()     → multiple documents (array)
findOne()  → one document by any condition
findById() → one document by _id
```

---

## Filtering

### 4. Comparison Operators

| Operator | Meaning | `.find({ price: { $gte: 500 } })` matches |
|---|---|---|
| `$gte` | `>=` | 500 ✅ 600 ✅ 1000 ✅ 499 ❌ |
| `$lte` | `<=` | 300 ✅ 500 ✅ 501 ❌ |
| `$gt`  | `>`  | 500 ❌ 501 ✅ 600 ✅ |
| `$lt`  | `<`  | 499 ✅ 500 ❌ 600 ❌ |

**Price range** — combine them:

```ts
const products = await Product.find({
  price: { $gte: 200, $lte: 500 }
});
// 200 <= price <= 500  → 200 ✅ 300 ✅ 500 ✅  199 ❌ 501 ❌
```

**Real API example:**

```http
GET /products?minPrice=200&maxPrice=500
```

```ts
interface PriceFilterQuery { minPrice?: string; maxPrice?: string }

app.get("/products", async (req, res) => {
  const { minPrice, maxPrice }: PriceFilterQuery = req.query;

  const filter: FilterQuery<Product> = {};
  if (minPrice || maxPrice) {
    filter.price = {};
    if (minPrice) filter.price.$gte = Number(minPrice);
    if (maxPrice) filter.price.$lte = Number(maxPrice);
  }

  const products = await Product.find(filter);
  res.json(products);
});
```

### 5. `$in`

Match **one of several values** (array membership).

```ts
const products = await Product.find({
  category: { $in: ["Food", "Toys"] }
});
// category = "Food" OR category = "Toys"
// Food ✅  Toys ✅  Medicine ❌
```

Useful when the frontend sends comma-separated filters:

```text
categories = Food,Toys
```

```ts
const categories = String(req.query.categories ?? "").split(",").filter(Boolean);
const products = await Product.find({ category: { $in: categories } });
```

> The inverse is `$nin`. Use `$in`/`$nin` only for **value** matching; for existence use `$exists`.

### 6. `$or` and `$and`

- `$or`  → **at least one** condition is true.
- `$and` → **all** conditions must be true.

```ts
// OR: category = Food  OR  price <= 200
const either = await Product.find({
  $or: [
    { category: "Food" },
    { price: { $lte: 200 } }
  ]
});

// AND: category = Food AND price >= 200
const both = await Product.find({
  category: "Food",
  price: { $gte: 200 }
});
```

MongoDB **implicitly ANDs** top-level fields, so you rarely write `$and` explicitly. You *do* need `$and` when you must repeat the same field with different conditions:

```ts
await Product.find({
  $and: [
    { tags: "special" },
    { tags: "new" }
  ]
});
```

The explicit equivalent of the implicit behavior above:

```ts
await Product.find({
  $and: [
    { category: "Food" },
    { price: { $gte: 200 } }
  ]
});
```

> Prefer the implicit (first) form for readability. Reach for explicit `$and` only when a plain object can't express the query (e.g. repeated keys).

---

## Query Result Control

Query builders are **chainable** and **lazy** — nothing hits the DB until you call `await`, `.exec()`, or a terminal method like `.countDocuments()`.

### 7. `sort()`

```ts
// ascending  (small → big)
const asc  = await Product.find().sort({ price: 1 });

// descending (big → small)
const desc = await Product.find().sort({ price: -1 });

// primary = price desc, secondary = name asc
const both = await Product.find().sort({ price: -1, name: 1 });
```

Multiple conditions: MongoDB sorts by the **first** key, then uses later keys as tie-breakers.

You can also pass a string or array form:

```ts
Product.find().sort("price -name");
Product.find().sort([["price", -1], ["name", 1]]);
```

> Sorting on an unindexed field forces an in-memory sort. Add an index (`Schema.index({ price: 1 })`) for large collections.

### 8. `limit()`

Cap the number of returned documents.

```ts
const products = await Product.find().limit(10); // at most 10, even with 500 in the DB
```

```ts
const limit = Math.min(Math.max(Number(req.query.limit) || 10, 1), 100); // clamp 1..100
const products = await Product.find().limit(limit);
```

Always clamp `limit` from user input — it's a common DoS vector.

### 9. `skip()` + Pagination

`skip(n)` ignores the first `n` documents; combined with `limit`, features the classic **offset pagination**.

```ts
const products = await Product.find().skip(10).limit(10); // docs 11 → 20
```

Page formula:

```ts
const page  = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 10;
const skip  = (page - 1) * limit;

const [products, total] = await Promise.all([
  Product.find().skip(skip).limit(limit),
  Product.countDocuments()
]);

res.json({
  data: products,
  pagination: {
    page,
    limit,
    total,
    totalPages: Math.ceil(total / limit),
    hasNextPage: page * limit < total,
    hasPrevPage: page > 1
  }
});
```

| Page | skip | limit | returns |
|---|---|---|---|
| 1 | 0  | 10 | 1 → 10 |
| 2 | 10 | 10 | 11 → 20 |
| 3 | 20 | 10 | 21 → 30 |

> **Performance warning:** offset pagination degrades on deep pages (`skip(100000)` scans ~100k docs). For infinite scroll / news feeds, prefer cursor-based pagination:

```ts
// cursor-based: WHERE price < lastPrice  (with index on price)
const before = req.query.before ? { price: { $lt: Number(req.query.before) } } : {};
const products = await Product.find(before).sort({ price: -1 }).limit(11);
const hasMore = products.length > 10;
```

---

## Relationships

### 10. `populate()`

MongoDB stores *references*, not JOIN tables. `populate()` resolves an `ObjectId` reference into the full referenced document — conceptually MongoDB's "JOIN".

```js
Product:  { _id: ..., name: "Dog Food", owner: ObjectId("123456") }
User:     { _id: "123456", name: "Ahmed" }
```

```ts
owner: { type: Schema.Types.ObjectId, ref: "User", required: true }
```

```ts
const product = await Product.findById(id).populate("owner");
```

Result — `owner` is now the full `User` document:

```js
{
  name: "Dog Food",
  owner: { _id: "123456", name: "Ahmed" }
}
```

**Advanced populate patterns:**

```ts
// 1. Pick fields
Product.find().populate("owner", "name email");

// 2. Nested populate (owner of owner)
Product.find().populate({ path: "owner", populate: { path: "company" } });

// 3. Populate multiple paths
Product.find().populate("owner").populate("category");

// 4. Fields + options (sort/limit on the populated side)
Product.find().populate({
  path: "owner",
  select: "name",
  options: { limit: 5, sort: { name: 1 } }
});

// 5. Conditional / match on populated docs
Product.find().populate({ path: "owner", match: { active: true } });
```

> `populate()` performs a **separate query per path** under the hood. It is not a single database JOIN — keep that in mind for heavy relational reads.

---

## Advanced Topics

### Projection — shape what comes back

```ts
const products = await Product.find({}, "name price");          // include
const products = await Product.find({}, { category: 0 });       // exclude
```

### `lean()` — skip Mongoose documents for raw speed

```ts
// plain JS objects, no hydration/change-tracking → ~2-3x faster reads
const products = await Product.find().lean();
```

Use `.lean()` for read-only endpoints. Keep Mongoose documents when you need virtuals, hooks, or later `.save()`.

### `select` vs `${getters}` and virtuals

```ts
productSchema.virtual("formattedPrice").get(function () {
  return `$${this.price.toFixed(2)}`;
});
// note: lean() drops virtuals unless you pass { virtuals: true }
```

### Bulk writes

```ts
const ops = Product.find({ category: "Food" }).map(p => ({
  updateOne: { filter: { _id: p._id }, update: { $inc: { price: 10 } } }
}));
await Product.bulkWrite(ops);
```

### Transactions (multi-collection consistency)

```ts
const session = await mongoose.startSession();
session.startTransaction();
try {
  await Order.create([{ ... }], { session });
  await Product.updateOne({ _id }, { $inc: { stock: -1 } }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

### Indexing for the queries you write

```ts
ProductSchema.index({ category: 1, price: -1 });   // supports {category, price} filters + sorts
ProductSchema.index({ createdAt: -1 });
```

Check your actual plans with:

```ts
await Product.find({ category: "Food" }).explain("executionStats");
```

---

## Repository Structure & Branches

Each topic is isolated on its own branch, with a deep-dive doc under `docs/`.

```
main
├── README.md                       ← this guide (everything combined)
└── docs/
    ├── 01-find.md
    ├── 02-find-one.md
    ├── 03-find-by-id.md
    ├── 04-comparison-operators.md
    ├── 05-in-operator.md
    ├── 06-or-and-operators.md
    ├── 07-sort.md
    ├── 08-limit.md
    ├── 09-skip-pagination.md
    └── 10-populate.md
```

| Branch | Topic |
|---|---|
| `main` | Everything |
| `feature/01-find` | `find()` retrieval |
| `feature/02-findOne` | `findOne()` retrieval |
| `feature/03-findById` | `findById()` retrieval |
| `feature/04-comparison-operators` | `$gte` `$lte` `$gt` `$lt` |
| `feature/05-in-operator` | `$in` |
| `feature/06-or-and-operators` | `$or` `$and` |
| `feature/07-sort` | `sort()` |
| `feature/08-limit` | `limit()` |
| `feature/09-skip-pagination` | `skip()` + pagination |
| `feature/10-populate` | `populate()` relationships |

**Explore a single topic:**

```bash
git fetch origin
git checkout -b feature/05-in-operator origin/feature/05-in-operator
```

---

## Cheat Sheet

```ts
// READ
Product.find({})                          // array
Product.findOne({ name: "x" })            // doc | null
Product.findById(id)                      // doc | null
Product.findByIdAndUpdate(id, upd, { new: true })
Product.findByIdAndDelete(id)

// FILTER
{ price: { $gte: 200, $lte: 500 } }
{ category: { $in: ["Food", "Toys"] } }
{ $or: [{ category: "Food" }, { price: { $lte: 200 } }] }
{ $and: [{ tags: "a" }, { tags: "b" }] }

// CONTROL
.find().sort({ price: -1 }).skip(10).limit(10).lean()

// RELATION
.findById(id).populate("owner", "name")
```

---

## Common Pitfalls

1. **`find()` returns an array** — treat it as `[]`, not `null`. Check `.length`, not existence truthiness.
2. **`findOne()` / `findById()` return `null`** — always null-check before touching fields.
3. **Query builders are lazy** — forgetting `await`/`.exec()` gives you a Query object, not data.
4. **Unclamped `limit`/`page` from query strings** — validate inputs to avoid slow/broken requests.
5. **Deep offset pagination** — `skip(100000)` is slow; switch to cursors for huge sets.
6. **Populate is N+1** — it issues extra queries per path; keep it in mind for heavy relational reads.
7. **`$and` vs `$or` mix-ups** — `$or` = any, all else is AND by default.
8. **Sorting unindexed fields** — forces an in-memory sort over the whole collection; add indexes in the schema.
9. **IDs are strings, not `ObjectId`(**)** — invalid IDs make `findById` throw a `CastError`; validate or catch it.

```text
Questions? Open an issue or a PR. Happy querying!
```