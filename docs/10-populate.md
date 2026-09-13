# Topic 10 — `populate()`

Resolve a stored `ObjectId` reference into the **referenced document**.

```js
// Product
{ name: "Dog Food", owner: ObjectId("123456") }

// User referenced by owner
{ _id: ObjectId("123456"), name: "Ahmed" }
```

## Schema side

```ts
owner: {
  type: mongoose.Schema.Types.ObjectId,
  ref: "User"          // ← must match the model name
}
```

## Populate

```ts
const product = await Product.findById(id).populate("owner");
```

Instead of `owner: "123456"` you now get the full user:

```js
{
  name: "Dog Food",
  owner: { _id: "123456", name: "Ahmed" }
}
```

## Advanced patterns

```ts
// pick only some fields
Product.find().populate("owner", "name email");

// nested populate (user → company)
Product.find().populate({ path: "owner", populate: { path: "company" } });

// multiple paths at once
Product.find().populate("owner").populate("category");

// constrain the populated docs
Product.find().populate({ path: "owner", match: { active: true } });

// sort/limit the populated results
Product.find().populate({
  path: "owner",
  select: "-passwordHash",
  options: { limit: 5, sort: { name: 1 } }
});
```

## How it works under the hood

`populate()` issues **one extra query per path** after the main query returns:

```text
1. db.products.find(...)
2. db.users.find({ _id: { $in: [ObjectId("123456"), ...] } })
```

This is the classic **N+1** in disguise: lending the term from SQL, populate is *"a JOIN done in application code"*.

## Gotchas

- `ref` string must exactly match the registered mongoose model name.
- Invalid IDs inside the array resolve to `null` in the populated path.
- No such feature as `populate` on plain Mongo driver — it is a Mongoose convenience.