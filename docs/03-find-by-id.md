# Topic 3 — `findById()`

Find a document **by its `_id`**.

```ts
const product = await Product.findById("64f123abc...");
```

Effectively `findOne({ _id: id })` — but rejects (throws `CastError`) when given a malformed ID.

```ts
const invalid = await Product.findById("not-an-objectid");
// throws: CastError: Cast to ObjectId failed
```

Always validate / wrap with try-catch:

```ts
app.get("/products/:id", async (req, res) => {
  if (!mongoose.isValidObjectId(req.params.id)) {
    return res.status(400).json({ message: "Invalid id" });
  }
  const product = await Product.findById(req.params.id);
  if (!product) return res.status(404).json({ message: "Not found" });
  res.json(product);
});
```

## Common variants

```ts
// find + update in one round-trip
const updated = await Product.findByIdAndUpdate(id, { price: 600 }, { new: true });

// find + delete
const removed = await Product.findByIdAndDelete(id);

// wait for full save lifecycle instead
await Product.updateOne({ _id: id }, { price: 600 });
```

## Summary

```text
find()        → multiple docs by criteria        (array)
findOne()     → one doc by criteria              (doc | null)
findById()    → one doc by _id                   (doc | null)
findByIdAndUpdate / findByIdAndDelete → mutation helpers
```