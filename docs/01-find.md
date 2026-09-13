# Topic 1 — `find()`

Retrieve **multiple documents** from a collection.

```ts
// All products
const products = await Product.find();

// Filter by category
const food = await Product.find({ category: "Food" });
```

`find()` **always** returns an array — even when zero or one document matches.

```text
find()  →  [] | [doc, doc, ...]
```

## Query builder / cursor form

Mongoose queries are **lazy query builders** — you can chain methods without hitting the DB:

```ts
const products = await Product
  .find({ category: "Food" })
  .sort({ price: 1 })
  .limit(5)
  .lean();
```

Equivalently with raw Mongo driver style:

```ts
const products = await Product.find({ category: "Food" }).exec();
```

## Chaining

```ts
const products = await Product.find({ category: "Toys" });
// "Dog Toy", "Fish Tank"
```

## Key takeaway

| Method | Return type | Null possible? |
|---|---|---|
| `find()` | Array | No |
| `findOne()` | Document | Yes |
| `findById()` | Document | Yes |