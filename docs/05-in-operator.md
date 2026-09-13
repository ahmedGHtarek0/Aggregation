# Topic 5 — `$in` (and `$nin`)

Match a field against **one of several values**.

```ts
const products = await Product.find({
  category: { $in: ["Food", "Toys"] }
});
// category = "Food"  OR  category = "Toys"
// Food ✅   Toys ✅   Medicine ❌
```

`$in` accepts an array of values, objects, or even embedded documents / `_id`s:

```ts
await Product.find({ _id: { $in: ids } });            // ObjectId membership
await Product.find({ tag: { $in: [/^new/i, "sale"] } });
```

## Real API example — comma-separated filter

```text
GET /products?categories=Food,Toys
```

```ts
app.get("/products", async (req, res) => {
  const categories = String(req.query.categories ?? "")
    .split(",")
    .map(s => s.trim())
    .filter(Boolean);

  const products = await Product.find({ category: { $in: categories } });
  res.json(products);
});
```

## Inverse: `$nin`

```ts
const products = await Product.find({
  category: { $nin: ["Medicine"] }
});
// excludes Medicine
```

| Operator | Matches if |
|---|---|
| `$in` | value equals **any** element |
| `$nin` | value equals **no** element |

> Prefer `$in` over chained `$or` for the same field — it is shorter and more index-friendly.

## Use it with `.countDocuments()`

```ts
const count = await Product.countDocuments({ category: { $in: ["Food", "Toys"] } });
```