# Smart collections, charts & routes — forest-express (legacy)

## Smart collection

Declare a collection that has no database table — you supply the records via a route.

```javascript
// /forest/customer_stats.js
const { collection } = require('forest-express-sequelize');

collection('customer_stats', {
  isSearchable: true,            // shows the search bar — you implement search in the route
  fields: [
    { field: 'id',           type: 'Number' },   // a unique id field is mandatory
    { field: 'email',        type: 'String' },
    { field: 'orders_count', type: 'Number' },
    { field: 'total_amount', type: 'Number' },
  ],
});
```

Implement a `GET /forest/customer_stats` route that builds the rows and serializes them (see below). Honor pagination via `request.query.page.size` / `page.number` and `request.query.search`.

## Serializing records — `RecordSerializer`

```javascript
const { RecordSerializer } = require('forest-express-sequelize');

const serializer = new RecordSerializer({ name: 'customer_stats' });
const payload = await serializer.serialize(records, { count: total });
response.send(payload);   // JSON:API { data: [...], meta: { count } }
```

## API-based charts

POST route returning a serialized stat. Use `StatSerializer`.

```javascript
const Liana = require('forest-express-sequelize');

// Value (single number)
router.post('/stats/mrr', (req, res) => {
  res.send(new Liana.StatSerializer({ value: mrr }).perform());
});

// Repartition (pie)
router.post('/stats/by-country', (req, res) => {
  res.send(new Liana.StatSerializer({
    value: [{ key: 'US', value: 42 }, { key: 'FR', value: 18 }],
  }).perform());
});

// Time-based (line)
router.post('/stats/charges-per-day', (req, res) => {
  res.send(new Liana.StatSerializer({
    value: [{ label: '2018-03-01', values: { value: 10 } }],
  }).perform());
});

// Objective
router.post('/stats/goal', (req, res) => {
  res.send(new Liana.StatSerializer({ value: { value: 1250, objective: 1500 } }).perform());
});
```

> A **Smart Chart** is a frontend component; the backend part is just a route returning the data it fetches (e.g. `GET /forest/custom-data`).

## Default routes

Auto-generated per collection in `/routes/<collection>.js`:

| Route | Middleware |
|-------|-----------|
| `POST /<c>` | `permissionMiddlewareCreator.create()` |
| `PUT /<c>/:recordId` | `.update()` |
| `DELETE /<c>` and `/<c>/:recordId` | `.delete()` |
| `GET /<c>` and `/<c>/count` | `.list()` |
| `GET /<c>/:recordId` | `.details()` |
| `GET /<c>.csv` | `.export()` |

## Extend a route — keep default behavior

Add logic, then call `next()` to run Forest's default handler.

```javascript
router.post('/customers', permissionMiddlewareCreator.create(), (req, res, next) => {
  // your logic (e.g. call an external API) ...
  next();   // → Forest's default create
});
```

## Override a route — replace default behavior

Omit `next()`; build and send the response yourself.

```javascript
const { RecordSerializer } = require('forest-express-sequelize');
const axios = require('axios');

router.post('/users', permissionMiddlewareCreator.create(), (request, response, next) => {
  const serializer = new RecordSerializer({ name: 'users' });
  axios.post('https://<api>/users', request.body.data.attributes)
    .then(r => serializer.serialize(r.data))
    .then(s => response.send(s))
    .catch(next);
});
```

## High-level record classes

`RecordGetter`, `RecordsGetter`, `RecordsCounter`, `RecordCreator`, `RecordUpdater`, `RecordRemover`, `RecordsRemover`, `RecordsExporter` — each `new X(model, request.user, request.query)`. They wrap the ORM and apply Forest scopes/filters.
