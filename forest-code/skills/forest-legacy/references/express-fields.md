# Smart fields, segments & relationships — forest-express (legacy)

Declared in `/forest/<collection>.js`. Import `collection` from `forest-express-sequelize` or `forest-express-mongoose`.

## Smart field

```javascript
const { collection } = require('forest-express-sequelize');

collection('customers', {
  fields: [
    {
      field: 'fullname',
      type: 'String',          // Boolean, Date, Dateonly, Enum, File, Number, Json, String, ['String']
      get: (customer) => `${customer.firstname} ${customer.lastname}`,
    },
  ],
});
```

| Option | Notes |
|--------|-------|
| `get(record)` | Compute the displayed value (may be async; can query related data) |
| `set(record, value)` | Make it writable — mutate and **return the record** |
| `search(query, search)` | Make it searchable — mutate and return the query |
| `filter({ condition, where })` | Make it filterable — return a where clause; needs `isFilterable: true` |
| `reference` | Turn it into a smart relationship (`'targetCollection.id'`) |
| `isFilterable`, `isSortable`, `isRequired`, `isReadOnly`, `enums`, `description` | |

```javascript
// Writable
set: (customer, fullname) => {
  const [first, last] = fullname.split(' ');
  customer.firstname = first;
  customer.lastname = last;
  return customer;
}

// Searchable (Sequelize)
search: (query, search) => {
  const { Op } = models.Sequelize;
  query.where[Op.and][0][Op.or].push({ firstname: { [Op.like]: `%${search}%` } });
  return query;
}

// Filterable
filter: ({ condition, where }) => {
  const { Op } = models.Sequelize;
  switch (condition.operator) {
    case 'equal': return { firstname: condition.value };
    case 'ends_with': return { firstname: { [Op.like]: `%${condition.value}` } };
    default: return null;
  }
}
```

## Smart segment

```javascript
const models = require('../models');
const { Op, QueryTypes } = models.objectMapping;

collection('products', {
  segments: [
    {
      name: 'Bestsellers',
      where: () =>
        models.connections.default
          .query('SELECT id FROM products JOIN orders ON orders.product_id = products.id GROUP BY products.id ORDER BY COUNT(orders.*) DESC LIMIT 5', { type: QueryTypes.SELECT })
          .then(rows => ({ id: { [Op.in]: rows.map(r => r.id) } })),
    },
  ],
});
```

Mongoose: return `{ _id: { $in: [...] } }` from an aggregation.

## Smart relationship

### belongsTo — a smart field with `reference` resolved in `get`

```javascript
collection('orders', {
  fields: [
    {
      field: 'delivery_address',
      type: 'String',
      reference: 'addresses.id',
      get: (order) =>
        models.addresses.findOne({ where: { id: order.customer_id } }),
    },
  ],
});
```

### hasMany — declare the field, then implement the relationship route

```javascript
// /forest/products.js
collection('products', {
  fields: [{ field: 'buyers', type: ['String'], reference: 'customers.id' }],
});
```

```javascript
// /routes/products.js — GET /products/:id/relationships/buyers
const { RecordSerializer } = require('forest-express-sequelize');
const { customers, orders } = require('../models');

router.get('/products/:product_id/relationships/buyers', async (request, response, next) => {
  try {
    const limit  = parseInt(request.query.page.size, 10) || 20;
    const offset = (parseInt(request.query.page.number, 10) - 1) * limit;
    const include = [{ model: orders, as: 'orders', where: { product_id: request.params.product_id } }];

    const [records, count] = await Promise.all([
      customers.findAll({ include, offset, limit }),
      customers.count({ include }),
    ]);

    const serialized = await new RecordSerializer(customers).serialize(records, { count });
    response.send(serialized);
  } catch (e) { next(e); }
});
```

### Smart action on a hasMany smart relationship — custom `getIdsFromRequest`

When a smart action targets a smart hasMany, implement id resolution yourself to honor "select all". Read `all_records`, `all_records_ids_excluded`, `all_records_subset_query`, `parent_collection_id`, `ids` from `request.body.data.attributes` and query the related ids; otherwise return `ids`.
