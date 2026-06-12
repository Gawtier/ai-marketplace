# Hooks & CRUD overrides — Ruby

## Hook basics

```ruby
collection.add_hook(
  'Before' | 'After',
  'List' | 'Create' | 'Update' | 'Delete' | 'Aggregate',
  ->(context) { ... }
)
```

---

## Hook types — when they fire and what's available

### Before Create

Modify `context.data` in place to change what gets saved. Raise to block.

```ruby
collection.add_hook('Before', 'Create', ->(context) {
  context.data.each do |record|
    record['created_by'] = context.caller.id
    record['status']   ||= 'pending'
  end
})
```

Block with a validation error:
```ruby
collection.add_hook('Before', 'Create', ->(context) {
  context.data.each do |record|
    unless record['email']&.include?('@')
      raise 'Invalid email address'
    end
  end
})
```

### Before Update

`context.patch` contains incoming changes. `context.filter` identifies which records are affected.

```ruby
collection.add_hook('Before', 'Update', ->(context) {
  context.patch['updated_by'] = context.caller.id
  context.patch['updated_at'] = Time.current
})
```

Block conditionally:
```ruby
collection.add_hook('Before', 'Update', ->(context) {
  if context.patch.key?('email')
    records = context.collection.list(context.filter, ['id', 'role'])
    locked = records.select { |r| r['role'] == 'superadmin' }
    raise 'Cannot change email of a superadmin' if locked.any?
  end
})
```

### Before Delete

```ruby
collection.add_hook('Before', 'Delete', ->(context) {
  records = context.collection.list(context.filter, ['id', 'status'])
  active  = records.select { |r| r['status'] == 'active' }
  raise "Cannot delete #{active.length} active record(s)" if active.any?
})
```

### After Create

`context.records` contains the created records with generated IDs.

```ruby
collection.add_hook('After', 'Create', ->(context) {
  context.records.each do |record|
    UserMailer.welcome(record['email']).deliver_later
    AuditLog.create!(
      event:     'created',
      record_id: record['id'],
      actor_id:  context.caller.id
    )
  end
})
```

### After Update

Use `context.collection.list` to fetch current state.

```ruby
collection.add_hook('After', 'Update', ->(context) {
  next unless context.patch.key?('status')

  records = context.collection.list(context.filter, ['id', 'email', 'status'])
  records.each do |record|
    NotificationService.status_changed(record['email'], context.patch['status'])
  end
})
```

### After Delete

```ruby
collection.add_hook('After', 'Delete', ->(context) {
  CleanupJob.perform_later(filter: context.filter.to_json)
  AuditLog.create!(event: 'deleted', actor_id: context.caller.id)
})
```

### Before/After List

Fires on every page load — use sparingly.

```ruby
collection.add_hook('Before', 'List', ->(context) {
  Rails.logger.info "[Forest] List by #{context.caller.email}"
})
```

---

## Context API — hooks

| Property | Available in | Description |
|----------|-------------|-------------|
| `context.data` | Before Create | Array of records to insert — modify in place |
| `context.patch` | Before Update | Hash of incoming changes |
| `context.filter` | Before/After all | Identifies affected records |
| `context.records` | After Create | Created records with generated IDs |
| `context.collection` | After all | Use `.list(filter, fields)` to fetch current state |
| `context.caller` | Before/After all | Current user |

---

## Fetching records in After hooks

```ruby
collection.add_hook('After', 'Update', ->(context) {
  records = context.collection.list(
    context.filter,
    ['id', 'email', 'first_name', 'status']
  )

  records.each { |record| process(record) }
})
```

---

## CRUD overrides

### override_create

Must return the created records array.

```ruby
collection.override_create do |context|
  created = context.data.map { |record| ExternalApi.create(record) }
  created  # must return array
end
```

### override_update

```ruby
collection.override_update do |context|
  ExternalApi.update(context.filter, context.patch)
  # no return value needed
end
```

### override_delete

```ruby
# Soft delete — set deleted_at instead of destroying
collection.override_delete do |context|
  records = Order.joins_filter(context.filter)
  records.update_all(
    deleted_at: Time.current,
    deleted_by: context.caller.id
  )
end
```

---

## Common patterns

### Audit log

```ruby
collection.add_hook('After', 'Create', ->(context) {
  AuditLog.insert_all(
    context.records.map do |r|
      { table: 'orders', record_id: r['id'], event: 'created', actor_id: context.caller.id }
    end
  )
})

collection.add_hook('After', 'Update', ->(context) {
  AuditLog.create!(
    table:    'orders',
    event:    'updated',
    changes:  context.patch.to_json,
    actor_id: context.caller.id
  )
})
```

### Default values on create

```ruby
collection.add_hook('Before', 'Create', ->(context) {
  context.data.each do |record|
    record['created_by'] = context.caller.id
    record['status']   ||= 'draft'
    record['team_id']  ||= context.caller.tags&.dig('team_id')
  end
})
```

### Notify on status change

```ruby
collection.add_hook('After', 'Update', ->(context) {
  next unless context.patch.key?('status')

  records = context.collection.list(context.filter, ['id', 'email', 'name'])
  records.each do |r|
    NotificationMailer.status_update(
      r['email'],
      r['name'],
      context.patch['status']
    ).deliver_later
  end
})
```
