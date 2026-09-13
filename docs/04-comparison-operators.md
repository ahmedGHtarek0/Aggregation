# Topic 4 — Comparison Operators

| Operator | Meaning | Example matches (against 500) |
|---|---|---|
| `$gte` | `>=` | 500 ✅ 600 ✅ 1000 ✅ 499 ❌ |
| `$lte` | `<=` | 300 ✅ 500 ✅ 501 ❌ |
| `$gt`  | `>`  | 500 ❌ 501 ✅ 600 ✅ |
| `$lt`  | `<`  | 499 ✅ 500 ❌ 600 ❌ |

## Individual usage

```ts
const expensive = await Product.find({ price: { $gte: 500 } });
const cheap     = await Product.find({ price: { $lte: 500 } });
const premium   = await Product.find({ price: { $gt: 500 } });
const budget    = await Product.find({ price: { $lt: 500 } });
```

## Price range (combined)

```ts
const products = await Product.find({
  price: { $gte: 200, $lte: 500 }
});
// 200 <= price <= 500
// 199 ❌   200 ✅   300 ✅   500 ✅   501 ❌
```

## API example

```http
GET /products?minPrice=200&maxPrice=500
```

```ts
app.get("/products", async (req, res) => {
  const { minPrice, maxPrice } = req.query;

  const filter: FilterQuery<Product> = {};
  if (minPrice ?? maxPrice) {
    filter.price = {};
    if (minPrice) filter.price.$gte = Number(minPrice);
    if (maxPrice) filter.price.$lte = Number(maxPrice);
  }

  const products = await Product.find(filter);
  res.json(products);
});
```

**Gotcha:** query string values are strings — always `Number(...)` them, and never put raw strings in comparison operators.