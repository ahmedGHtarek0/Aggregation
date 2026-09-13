# Topic 2 — `findOne()`

Retrieve **a single document** matching a condition.

```ts
const dogFood = await Product.findOne({ name: "Dog Food" });
const anyFood = await Product.findOne({ category: "Food" });
```

Even when many documents match, `findOne()` returns only the **first** (by natural/index order).

## Return values

```text
found     → the document
not found → null
```

Always null-check:

```ts
const product = await Product.findOne({ name: "Dog Food" });
if (!product) return res.status(404).json({ message: "Not found" });
```

## Real API pattern

```http
GET /products/search?name=Dog Food
```

```ts
app.get("/products/search", async (req, res) => {
  const product = await Product.findOne({ name: req.query.name });
  product ? res.json(product) : res.status(404).end();
});
```

## find() vs findOne()

```text
find()     → []   | [doc, doc]
findOne()  → null | doc
```

Use `find()` when the API is "list", `findOne()` when it is "get one by criteria".