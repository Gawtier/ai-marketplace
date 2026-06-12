# Fields & relationships — Ruby

## Computed field

### Synchronous

```ruby
collection.add_field('full_name', {
  column_type: 'String',
  dependencies: ['first_name', 'last_name'],
  # Receives ALL records on the current page — return one value per record, same order
  get_values: ->(records) {
    records.map { |r| "#{r['first_name']} #{r['last_name']}" }
  }
})
```

### Async — always batch

```ruby
collection.add_field('lifetime_revenue', {
  column_type: 'Number',
  dependencies: ['id'],
  get_values: ->(records) {
    # One call for all IDs — never loop with per-record queries
    ids      = records.map { |r| r['id'] }
    revenues = RevenueService.batch_fetch(ids)
    # revenues must be an array aligned with records (same length, same order)
    revenues
  }
})
```

### Computed from a relation

```ruby
collection.add_field('customer_country', {
  column_type: 'String',
  dependencies: ['customer:country'],  # access nested via relation:field
  get_values: ->(records) {
    records.map { |r| r.dig('customer', 'country') }
  }
})
```

### Enum computed field

```ruby
collection.add_field('risk_level', {
  column_type: 'Enum',
  enum_values: ['low', 'medium', 'high'],
  dependencies: ['score'],
  get_values: ->(records) {
    records.map do |r|
      score = r['score'].to_i
      if    score >= 80 then 'high'
      elsif score >= 40 then 'medium'
      else                   'low'
      end
    end
  }
})
```

**Column types:** `String` `Number` `Boolean` `Date` `Dateonly` `Time` `Enum` (add `enum_values:`) `Json` `Uuid` `Point` `Binary` `File`

---

## Write handler

Returns a **patch hash** — not the full record. Only include the fields to update.

```ruby
# Split a virtual full_name field into first_name + last_name
collection.replace_field_writing('full_name', ->(value, _context) {
  parts = value.split(' ', 2)
  { 'first_name' => parts[0], 'last_name' => parts[1] }
})
```

```ruby
# Convert a price_with_tax display field into price_without_tax stored field
collection.replace_field_writing('price_with_tax', ->(value, _context) {
  { 'price_without_tax' => (value / 1.2).round(2) }
})
```

Accessing the caller:
```ruby
collection.replace_field_writing('approved_by', ->(value, context) {
  { 'approved_by_id' => context.caller.id, 'approved_at' => Time.current }
})
```

---

## Validation

Chain multiple rules on the same field:

```ruby
collection
  .add_field_validation('email', 'Present')
  .add_field_validation('email', 'Match', /\A[\w.]+@[\w.]+\.\w{2,}\z/)
  .add_field_validation('username', 'LongerThan', 3)
  .add_field_validation('username', 'ShorterThan', 50)
  .add_field_validation('age', 'GreaterThan', 0)
  .add_field_validation('age', 'LessThan', 150)
  .add_field_validation('expires_at', 'After', Time.current)
```

**All operators:** `Present` `Blank` `GreaterThan` `LessThan` `LongerThan` `ShorterThan` `Contains` `Like` `Match` `Before` `After` `In` `NotIn`

---

## Import, rename, remove

```ruby
# Import a field from a relation
collection.import_field('customer_email', {
  path: 'customer:email',    # relation:field (can chain: 'customer:company:name')
  readonly: true
})

# Rename in the UI without changing the DB column
collection.rename_field('created_at', 'createdAt')
collection.rename_field('usr_id', 'userId')

# Remove from UI (field stays in DB, still usable in dependencies)
collection.remove_field('internal_notes', 'debug_payload', 'legacy_field')

# Binary mode: 'datauri' for images/files, 'hex' for short binary like UUIDs
collection.replace_field_binary_mode('avatar', 'datauri')
collection.replace_field_binary_mode('fingerprint', 'hex')
```

---

## Relationships

### Many-to-one

```ruby
# orders.customer_id → customers.id
ForestAdmin.agent.customize_collection('Order') do |collection|
  collection.add_many_to_one_relation('customer', 'Customer', {
    foreign_key: 'customer_id',
    foreign_key_target: 'id'   # optional, defaults to primary key
  })
end
```

### One-to-many

```ruby
# customers.id ← orders.customer_id
ForestAdmin.agent.customize_collection('Customer') do |collection|
  collection.add_one_to_many_relation('orders', 'Order', {
    origin_key: 'customer_id',
    origin_key_target: 'id'   # optional
  })
end
```

### One-to-one

```ruby
ForestAdmin.agent.customize_collection('Person') do |collection|
  collection.add_one_to_one_relation('profile', 'Profile', {
    origin_key: 'person_id',
    origin_key_target: 'id'   # optional, defaults to primary key
  })
end
```

### Many-to-many (through junction table)

```ruby
ForestAdmin.agent.customize_collection('Product') do |collection|
  collection.add_many_to_many_relation('tags', 'Tag', 'ProductTag', {
    origin_key: 'product_id',    # FK in junction pointing to this collection
    foreign_key: 'tag_id',       # FK in junction pointing to the target collection
    origin_key_target: 'id',     # optional
    foreign_key_target: 'id'     # optional
  })
end
```

### External / virtual relation

No FK — data fetched from an external source. Appears only in the "Related Data" panel.

> **External relationships do not support pagination.** `list_records` returns the full list.

`list_records` receives a `record` hash containing the **dependency fields** of the parent. `dependencies` is optional — by default only the primary key is provided.

```ruby
collection.add_external_relation('near_states',
  # Schema of the returned records
  schema: { 'code' => 'String', 'name' => 'String' },
  # Which parent fields the handler needs (optional; defaults to the primary key)
  dependencies: ['country', 'zip_code'],
  # record holds the requested parent fields — return an array matching the schema
  list_records: ->(record) {
    return [] unless record['country'] == 'USA'
    StateApi.near(record['zip_code'])
  }
)
```

---

## Computed FK relationships

When a FK isn't stored in the DB, add a computed field first, then use it as FK.

```ruby
ForestAdmin.agent.customize_collection('Log') do |collection|
  # Step 1 — add the computed FK field
  collection.add_field('user_id', {
    column_type: 'Number',
    dependencies: ['metadata'],
    get_values: ->(records) {
      records.map { |r| r.dig('metadata', 'user_id') }
    }
  })

  # Step 2 — declare the relationship using the computed field
  collection.add_many_to_one_relation('user', 'User', {
    foreign_key: 'user_id'
  })
end
```
