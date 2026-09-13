# Topic 9 — `skip()` + Pagination

`skip(n)` ignores the first `n` documents; `limit(n)` caps the page size.

```ts
const products = await Product.find().skip(10).limit(10); // docs 11 → 20
```

Skipped `1 → 10`, returned `11 → 20`.

## Page formula

```text
skip = (page - 1) * limit
```

```ts
const page  = Number(req.query.page) || 1;
const limit = Number(req.query.limit) || 10;
const skip  = (page - 1) * limit;
```

| Page | skip | limit | returns |
|---|---|---|---|
| 1 | 0  | 10 | 1 → 10 |
| 2 | 10 | 10 | 11 → 20 |
| 3 | 20 | 10 | 21 → 30 |

## Full paginated endpoint

```ts
app.get("/products", async (req, res) => {
  const page  = Math.max(Number(req.query.page) ?? 1, 1);
  const limit = Math.min(Math.max(Number(req.query.limit) ?? 10, 1), 100);
  const skip  = (page - 1) * limit;

  const [products, total] = await Promise.all([
    Product.find().skip(skip).limit(limit),
    Product.countDocuments()
  ]);

  res.json({
    data: products,
    page,
    limit,
    total,
    totalPages: Math.ceil(total / limit),
    hasNextPage: page * limit < total,
    hasPrevPage: page > 1
  });
});
```

## When offset pagination hurts

`skip(100000)` still scans ~100k documents — deep pages get slow. For infinite scroll / feeds, use **cursor-based** pagination driven by a monotonic indexed field:

```ts
// WHERE price < lastSeenPrice, ordered desc
const before = req.query.before
  ? { price: { $lt: Number(req.query.before) } }
  : {};

const products = await Product.find(before)
  .sort({ price: -1 })
  .limit(11);                     // fetch one extra to detect "has more"

const hasMore = products.length > 11;
const pageData = hasMore ? products.slice(0, 10) : products;
```

## Guideline

| Dataset / UX | Use |
|---|---|
| Admin tables, small datasets | offset (`skip`/`limit`) |
| Feeds, infinite scroll, huge data | cursor (keyset) |

> Always pair pagination with a deterministic `sort()` — otherwise pages can repeat or skip docs.