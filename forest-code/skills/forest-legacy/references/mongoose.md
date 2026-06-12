# Mongoose model layer — forest-express (legacy)

Stack: `forest-express-mongoose`. Models live in `/models/<name>.js`.

## Model definition

```javascript
module.exports = (mongoose, Mongoose) => {
  const schema = Mongoose.Schema({
    name: String,
    customer_id: { type: Mongoose.Schema.Types.ObjectId, ref: 'customers' }, // belongsTo
    orders: [{ type: Mongoose.Schema.Types.ObjectId, ref: 'orders' }],        // hasMany
  });
  return mongoose.model('companies', schema, 'companies');
};
```

Relationships are schema refs (no `.associate()`). `belongsToMany` has no built-in form — model it with an array of refs or an aggregation.

## Query idioms (inside routes, smart fields, actions)

```javascript
await customers.findById(id);
await visualizations.find({ user: userObjectId }, null, { skip: offset, limit });
await visualizations.countDocuments({ user: userObjectId });

// Joins via aggregation
await customers.aggregate([
  { $lookup: { from: 'orders', localField: 'orders', foreignField: '_id', as: 'orders' } },
  { $unwind: '$orders' },
  { $match: { 'orders.product_id': mongoose.Types.ObjectId(productId) } },
]);
```

Cast incoming ids: `mongoose.Types.ObjectId(request.params.recordId)`.

## Operators in smart-field search / filter

```javascript
// search: return a Mongoose filter object
search: (search) => {
  const [first, last] = search.split(' ');
  return { firstname: first, lastname: last };
}

// filter: return MongoDB operator syntax
filter: ({ condition }) => ({ firstname: { $regex: `.*${condition.value}` } })
```

## Smart hasMany in MongoDB (documented pattern)

```javascript
// /forest/user.js
collection('user', {
  fields: [{ field: 'visualizations', type: ['String'], reference: 'visualization._id' }],
});

// /routes/user.js — GET /user/:recordId/relationships/visualizations
router.get('/user/:recordId/relationships/visualizations', async (req, res) => {
  const userObjectId = mongoose.Types.ObjectId(req.params.recordId);
  const limit  = parseInt(req.query.page.size, 10) || 20;
  const offset = (parseInt(req.query.page.number, 10) - 1) * limit;
  const [data, count] = await Promise.all([
    visualization.find({ user: userObjectId }, null, { skip: offset, limit }),
    visualization.countDocuments({ user: userObjectId }),
  ]);
  res.send(await new RecordSerializer({ name: 'visualization' }).serialize(data, { count }));
});
```

## Batching related lookups

DataLoader keyed by id, resolving with a single `find({ _id: { $in: keys } })` (or `{ customer_id: { $in: keys } }`).
