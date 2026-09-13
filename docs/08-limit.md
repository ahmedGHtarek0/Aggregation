# Topic 8 — `limit()`

Return **at most** N documents.

```ts
const products = await Product.find().limit(10);
// 500 products in DB → returns ≤ 10
```

`limit(0)` in Mongoose means "no limit".

## Real API example

```text
GET /products?limit=10
```

```ts
app.get("/products", async (req, res) => {
  const limit = Number(req.query.limit) || 10;
  const products = await Product.find().limit(limit);
  res.json(products);
});
```

## Always clamp user input

Never trust raw query values — clamp them to a safe range:

```ts
const limit = Math.min(Math.max(Number(req.query.limit) || 10, 1), 100);
const products = await Product.find().limit(limit);
```

Return the effective limit alongside the data:

```ts
res.json({ limit, count: products.length, data: products });
```

## limit + sort

Combine with `sort()` so you always get the **same** docs on repeat calls:

```ts
// top 5 cheapest
const cheapest = await Product.find().sort({ price: 1 }).limit(5);

// top 10 most expensive
const top = await Product.find().sort({ price: -1 }).limit(10);
```

## Note

`limit()` caps output but does **not** tell you the total match count — that requires a separate `countDocuments()` (pair it with pagination).