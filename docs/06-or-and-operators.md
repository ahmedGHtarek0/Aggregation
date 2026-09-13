# Topic 6 — `$or` and `$and`

| Operator | Meaning |
|---|---|
| `$or`  | **at least one** condition must be true |
| `$and` | **all** conditions must be true |

## `$or`

```ts
const products = await Product.find({
  $or: [
    { category: "Food" },
    { price: { $lte: 200 } }
  ]
});
// category = "Food"  OR  price <= 200
```

Mix several fields freely:

```ts
await Product.find({
  $or: [
    { name: /dog/i },
    { category: "Toys" },
    { price: { $lt: 100 } }
  ]
});
```

## `$and`

MongoDB **implicitly ANDs** all top-level fields, so you write it rarely:

```ts
// this is already AND
await Product.find({ category: "Food", price: { $gte: 200 } });
```

The explicit form is equivalent:

```ts
await Product.find({
  $and: [
    { category: "Food" },
    { price: { $gte: 200 } }
  ]
});
```

Use explicit `$and` when you need to apply multiple conditions to the **same field**:

```ts
await Product.find({
  $and: [
    { tags: "special" },
    { tags: "sale" }
  ]
});
```

A plain object can't express that (keys would collide), but `$and` can.

## Combining them

```ts
await Product.find({
  $and: [
    { category: "Food" },
    {
      $or: [
        { price: { $gte: 500 } },
        { inStock: true }
      ]
    }
  ]
});
```

## Cheat sheet

| Statement | Returns if |
|---|---|
| `{ a: 1, b: 2 }` | a=1 **AND** b=2 |
| `{ $or: [...] }` | any condition true |
| `{ $and: [...] }` | all conditions true |