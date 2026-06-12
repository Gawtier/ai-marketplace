# Hooks & CRUD overrides — Node.js

## Hook basics

```typescript
collection.addHook(
  position: 'Before' | 'After',
  type: 'List' | 'Create' | 'Update' | 'Delete' | 'Aggregate',
  handler: async (context) => { ... }
)
```

---

## Hook types — when they fire and what's available

### Before Create

Fires before records are inserted. Modify `context.data` in place to change what gets saved. Throw to block the operation.

```typescript
collection.addHook('Before', 'Create', async (context) => {
  context.data.forEach(record => {
    record.createdBy = context.caller.id
    record.status  ??= 'pending'
    record.uuid      = crypto.randomUUID()
  })
})
```

Block with a validation error:
```typescript
collection.addHook('Before', 'Create', async (context) => {
  for (const record of context.data) {
    if (!record.email?.includes('@')) {
      throw new Error('Invalid email address')
    }
  }
})
```

### Before Update

Fires before records are patched. `context.patch` contains the incoming changes. `context.filter` identifies which records are affected.

```typescript
collection.addHook('Before', 'Update', async (context) => {
  // Add updatedBy to every update
  context.patch.updatedBy = context.caller.id
  context.patch.updatedAt = new Date()
})
```

Block conditionally:
```typescript
collection.addHook('Before', 'Update', async (context) => {
  if ('email' in context.patch) {
    const records = await context.collection.list(context.filter, ['id', 'role'])
    const locked = records.filter(r => r.role === 'superadmin')
    if (locked.length > 0) {
      throw new Error('Cannot change email of a superadmin')
    }
  }
})
```

### Before Delete

```typescript
collection.addHook('Before', 'Delete', async (context) => {
  const records = await context.collection.list(context.filter, ['id', 'status'])
  const active = records.filter(r => r.status === 'active')
  if (active.length > 0) {
    throw new Error(`Cannot delete ${active.length} active record(s)`)
  }
})
```

### After Create

Fires after records are inserted. `context.records` contains the created records.

```typescript
collection.addHook('After', 'Create', async (context) => {
  for (const record of context.records) {
    await sendWelcomeEmail(record.email)
    await createAuditEntry({ event: 'created', recordId: record.id, actor: context.caller.id })
  }
})
```

### After Update

Fires after records are patched. Use `context.collection.list` to fetch current state.

```typescript
collection.addHook('After', 'Update', async (context) => {
  if ('status' in context.patch) {
    const records = await context.collection.list(context.filter, ['id', 'email', 'status'])
    await Promise.all(
      records.map(r => notifyStatusChange(r.email, context.patch.status))
    )
  }
})
```

### After Delete

```typescript
collection.addHook('After', 'Delete', async (context) => {
  // context.filter identifies what was deleted
  await cleanupRelatedData(context.filter)
  await createAuditEntry({ event: 'deleted', actor: context.caller.id })
})
```

### Before/After List

Useful for logging or modifying filters. Use sparingly — fires on every page load.

```typescript
collection.addHook('Before', 'List', async (context) => {
  // Restrict visibility based on caller's team
  if (context.caller.team !== 'admin') {
    // context.filter can be extended here (advanced use case)
  }
})
```

---

## Context API — hooks

| Property | Available in | Description |
|----------|-------------|-------------|
| `context.data` | Before Create | Array of records to be inserted — modify in place |
| `context.patch` | Before Update | Patch object with incoming changes |
| `context.filter` | Before/After all | Identifies affected records (use with `collection.list`) |
| `context.records` | After Create | Created records with generated IDs |
| `context.collection` | After all | Use `.list(filter, fields)` to fetch current state |
| `context.caller` | Before/After all | Current user |

---

## Fetching records in After hooks

```typescript
collection.addHook('After', 'Update', async (context) => {
  // Always specify only the fields you need
  const records = await context.collection.list(
    context.filter,
    ['id', 'email', 'firstName', 'status']
  )

  for (const record of records) {
    await doSomething(record)
  }
})
```

---

## CRUD overrides

Overrides replace the default DB operation entirely — use when data lives outside the database or requires custom logic.

### overrideCreate

Must return the created records array (Forest Admin uses it to display the result).

```typescript
collection.overrideCreate(async (context) => {
  const createdRecords = await Promise.all(
    context.data.map(record => externalApi.create(record))
  )
  return createdRecords
})
```

### overrideUpdate

```typescript
collection.overrideUpdate(async (context) => {
  // context.filter identifies affected records
  // context.patch contains the changes
  await externalApi.update(context.filter, context.patch)
  // no return value needed
})
```

### overrideDelete

```typescript
// Soft delete example: set deletedAt instead of removing from DB
collection.overrideDelete(async (context) => {
  const records = await collection.list(context.filter, ['id'])
  await db.update('orders')
    .set({ deletedAt: new Date(), deletedBy: context.caller.id })
    .whereIn('id', records.map(r => r.id))
})
```

---

## Common patterns

### Audit log

```typescript
collection.addHook('After', 'Create', async (context) => {
  await AuditLog.insertMany(
    context.records.map(r => ({
      table:     'orders',
      recordId:  r.id,
      event:     'created',
      actorId:   context.caller.id,
      timestamp: new Date(),
    }))
  )
})

collection.addHook('After', 'Update', async (context) => {
  await AuditLog.insert({
    table:   'orders',
    event:   'updated',
    changes: context.patch,
    actorId: context.caller.id,
  })
})
```

### Default values on create

```typescript
collection.addHook('Before', 'Create', async (context) => {
  context.data.forEach(record => {
    record.createdBy  = context.caller.id
    record.createdAt  = new Date()
    record.status   ??= 'draft'
    record.teamId   ??= context.caller.tags?.teamId
  })
})
```

### Notify on status change

```typescript
collection.addHook('After', 'Update', async (context) => {
  if (!('status' in context.patch)) return

  const records = await context.collection.list(context.filter, ['id', 'email', 'name'])
  await Promise.all(
    records.map(r =>
      NotificationService.sendStatusUpdate(r.email, r.name, context.patch.status)
    )
  )
})
```
