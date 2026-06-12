# Fields & relationships — Node.js

## Computed field

### Synchronous

```typescript
collection.addField('fullName', {
  columnType: 'String',
  dependencies: ['firstName', 'lastName'],
  // Receives ALL records on the current page — return one value per record, same order
  getValues: (records) => records.map(r => `${r.firstName} ${r.lastName}`),
})
```

### Async — always batch

```typescript
collection.addField('lifetimeRevenue', {
  columnType: 'Number',
  dependencies: ['id'],
  getValues: async (records) => {
    // One call for all IDs — never loop with per-record queries
    const ids = records.map(r => r.id)
    const revenues = await RevenueService.batchFetch(ids)
    // revenues must be an array aligned with records (same length, same order)
    return revenues
  },
})
```

### Computed from a relation

```typescript
// Access a related field via its relation name (requires the relation to be defined)
collection.addField('customerCountry', {
  columnType: 'String',
  dependencies: ['customer:country'],  // dot path through the relation
  getValues: (records) => records.map(r => r.customer?.country ?? null),
})
```

### Computed from another computed field

```typescript
// firstName and fullName must already be declared before this one
collection.addField('initials', {
  columnType: 'String',
  dependencies: ['firstName', 'lastName'],
  getValues: (records) =>
    records.map(r =>
      `${r.firstName?.[0] ?? ''}${r.lastName?.[0] ?? ''}`.toUpperCase()
    ),
})
```

### Enum computed field

```typescript
collection.addField('riskLevel', {
  columnType: 'Enum',
  enumValues: ['low', 'medium', 'high'],
  dependencies: ['score'],
  getValues: (records) =>
    records.map(r => {
      if (r.score >= 80) return 'high'
      if (r.score >= 40) return 'medium'
      return 'low'
    }),
})
```

**Column types:** `String` `Number` `Boolean` `Date` `Dateonly` `Time` `Enum` (add `enumValues`) `Json` `Uuid` `Point` `Binary` `File` — arrays: `['String']` `['Number']` `['Enum']`

---

## Write handler

Returns a **patch object** — not the full record. Only include the fields to update.

```typescript
// Split a virtual fullName field into firstName + lastName
collection.replaceFieldWriting('fullName', (value, context) => {
  const [firstName, ...rest] = value.split(' ')
  return { firstName, lastName: rest.join(' ') }
})
```

```typescript
// Convert a price_with_tax display field into price_without_tax stored field
collection.replaceFieldWriting('priceWithTax', (value, context) => ({
  priceWithoutTax: Math.round(value / 1.2 * 100) / 100,
}))
```

Accessing the caller inside a write handler:
```typescript
collection.replaceFieldWriting('approvedBy', (value, context) => ({
  approvedById: context.caller.id,
  approvedAt: new Date(),
}))
```

---

## Validation

Chain multiple rules on the same field:

```typescript
collection
  .addFieldValidation('email', 'Present')
  .addFieldValidation('email', 'Match', /^[\w.]+@[\w.]+\.\w{2,}$/)
  .addFieldValidation('username', 'LongerThan', 3)
  .addFieldValidation('username', 'ShorterThan', 50)
  .addFieldValidation('age', 'GreaterThan', 0)
  .addFieldValidation('age', 'LessThan', 150)
  .addFieldValidation('expiresAt', 'After', new Date())
```

**All operators:** `Present` `Blank` `GreaterThan` `LessThan` `LongerThan` `ShorterThan` `Contains` `Like` `Match` `Before` `After` `In` `NotIn`

---

## Import, rename, remove

```typescript
// Import a field from a relation — makes it appear directly on the collection
collection.importField('customerEmail', {
  path: 'customer:email',   // relation:field (can chain: 'customer:company:name')
  readonly: true,
})

// Rename in the UI without changing the DB column
collection.renameField('created_at', 'createdAt')
collection.renameField('usr_id', 'userId')

// Remove from UI (field stays in DB, still usable in dependencies)
collection.removeField('internalNotes', 'debugPayload', 'legacyField')

// Mark a required DB field as optional in the UI
collection.setFieldNullable('middleName')

// Binary mode: 'datauri' for images/files, 'hex' for short binary like UUIDs
collection.replaceFieldBinaryMode('avatar', 'datauri')
collection.replaceFieldBinaryMode('fingerprint', 'hex')
```

---

## Relationships

### Many-to-one

```typescript
// orders.customerId → customers.id
agent.customizeCollection('orders', collection => {
  collection.addManyToOneRelation('customer', 'customers', {
    foreignKey: 'customerId',
    foreignKeyTarget: 'id',  // optional, defaults to primary key
  })
})
```

### One-to-many

```typescript
// customers.id ← orders.customerId
agent.customizeCollection('customers', collection => {
  collection.addOneToManyRelation('orders', 'orders', {
    originKey: 'customerId',
    originKeyTarget: 'id',  // optional
  })
})
```

### One-to-one

```typescript
agent.customizeCollection('persons', collection => {
  collection.addOneToOneRelation('profile', 'profiles', {
    originKey: 'personId',
    originKeyTarget: 'id',  // optional, defaults to primary key
  })
})
```

### Many-to-many (through junction table)

```typescript
agent.customizeCollection('products', collection => {
  collection.addManyToManyRelation('tags', 'tags', 'product_tags', {
    originKey: 'productId',       // FK in junction pointing to this collection
    foreignKey: 'tagId',          // FK in junction pointing to the target collection
    originKeyTarget: 'id',        // optional
    foreignKeyTarget: 'id',       // optional
  })
})
```

### External / virtual relation

No FK — data is fetched from an external source (API, another DB, etc.).
Appears only in the "Related Data" panel of each record.

> **External relationships do not support pagination.** `listRecords` returns the full list.

`listRecords` receives the **dependency fields** of the parent record (destructured), not a wrapper object. `dependencies` is optional — by default only the primary key is provided.

```typescript
collection.addExternalRelation('nearStates', {
  // Schema of the returned records
  schema: { code: 'String', name: 'String' },
  // Which parent fields the handler needs (optional; defaults to the primary key)
  dependencies: ['country', 'zipCode'],
  // Receives the requested parent fields — return an array matching the schema
  listRecords: async ({ country, zipCode }) => {
    if (country !== 'USA') return []
    return StateApi.near(zipCode)
  },
})
```

---

## Computed FK relationships

When a FK isn't stored in the DB, add a computed field first, then use it as a FK.

```typescript
agent.customizeCollection('logs', collection => {
  // Step 1 — add the computed FK field
  collection.addField('userId', {
    columnType: 'Number',
    dependencies: ['metadata'],
    getValues: (records) => records.map(r => r.metadata?.userId ?? null),
  })

  // Step 2 — declare the relationship using the computed field
  collection.addManyToOneRelation('user', 'users', {
    foreignKey: 'userId',
  })
})
```
