# Smart actions — forest-express (legacy)

Import from `forest-express-sequelize` or `forest-express-mongoose` depending on the stack — the action API is identical.

## Declaration — `/forest/<collection>.js`

```javascript
const { collection } = require('forest-express-sequelize');

collection('companies', {
  actions: [
    {
      name: 'Mark as Live',     // label shown in the UI
      type: 'single',           // 'single' | 'bulk' | 'global'
      fields: [],
      download: false,          // true → response is a file download
      endpoint: '/forest/actions/mark-as-live',  // optional, defaults to dasherized name
      httpMethod: 'POST',
    },
  ],
});
```

## Form fields

```javascript
fields: [
  {
    field: 'amount',
    type: 'Number',            // String, Number, Boolean, Date, Dateonly, Enum, File, ['String']
    isRequired: true,
    isReadOnly: false,
    description: 'Amount in USD',
    defaultValue: 100,
    enums: ['US', 'FR', 'DE'], // for type 'Enum'
    reference: 'companies.id',  // record picker into another collection
    widgetEdit: 'text area editor', // 'boolean editor', 'date editor', 'file picker', ...
    hook: 'onAmountChange',     // name of a change hook (see below)
  },
]
```

## Dynamic forms — `hooks`

```javascript
actions: [
  {
    name: 'Send invoice',
    type: 'single',
    fields: [/* ... */],
    hooks: {
      // Runs when the form opens — mutate and RETURN fields
      load: async ({ fields, request }) => {
        const country = fields.find(f => f.field === 'country');
        country.value = 'France';
        return fields;
      },
      // Keyed by the field's `hook` name — runs when that field changes
      change: {
        onCityChange: async ({ fields, request, changedField }) => {
          const zip = fields.find(f => f.field === 'zip code');
          zip.value = getZipFromCity(changedField.value);
          return fields;
        },
      },
    },
  },
]
```

Hook context: `fields` (array, must be returned), `request`, `changedField` (change hooks only). Add fields dynamically with `fields.push({ field, type })`.

## Route handler — `/routes/<collection>.js`

```javascript
const express = require('express');
const { PermissionMiddlewareCreator, RecordsGetter } = require('forest-express-sequelize');
const { companies } = require('../models');

const router = express.Router();
const permissionMiddlewareCreator = new PermissionMiddlewareCreator('companies');

router.post(
  '/actions/mark-as-live',
  permissionMiddlewareCreator.smartAction(),
  async (request, response) => {
    // Selected record ids — handles "select all" / filters / exclusions
    const ids = await new RecordsGetter(companies, request.user, request.query)
      .getIdsFromRequest(request);

    // Form values
    const { amount } = request.body.data.attributes.values;

    await companies.update({ status: 'live' }, { where: { id: ids } });

    response.send({ success: 'Company is now live!' });
  },
);

module.exports = router;
```

## Request payload — `request.body.data.attributes`

| Field | Meaning |
|-------|---------|
| `ids` | Selected record ids |
| `values` | Form field values (keyed by field name) |
| `all_records`, `all_records_ids_excluded`, `all_records_subset_query` | "Select all" state — let `getIdsFromRequest` resolve it |
| `collection_name`, `parent_collection_name`, `parent_collection_id`, `parent_association_name` | Context |
| `action_intent_params` | Params passed via an action-intent URL |

`request.user` holds the current Forest user (`id`, `email`, `firstName`, `lastName`, `role`, `team`, `tags`).

## Responses

| Status | Body | Effect |
|--------|------|--------|
| `204` | (empty) | Default success toast |
| `200` | `{ success: 'msg' }` | Custom success toast |
| `200` | `{ html: '<p>…</p>' }` | HTML shown to the operator |
| `200` | `{ success, refresh: { relationships: ['rel'] } }` | Refresh related data |
| `200` | `{ success, redirectTo: 'url-or-internal-path' }` | Redirect |
| `200` | `{ webhook: { url, method, headers, body } }` | Fire a webhook |
| `400` | `{ error: 'msg' }` | Error toast |

## Action intent

Pre-fill / pre-trigger an action via URL:

```
…/data/<collection>/index?actionIntent=<name>&actionIntentIds=[1,2]&actionIntentParams={"key":"value"}
```

Read in a hook via `request.body.data.attributes.action_intent_params`.
