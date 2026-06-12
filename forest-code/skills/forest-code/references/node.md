# Forest Admin — Node.js general reference

Segments, charts, search, filtering, sorting, and misc. For actions see `action-node.md`, for hooks/overrides see `hook-node.md`, for fields/relationships see `field-node.md`.

---

## Segment

```typescript
// Static
collection.addSegment('VIP customers', {
  aggregator: 'And',
  conditions: [
    { field: 'status',         operator: 'Equal',       value: 'active' },
    { field: 'lifetimeValue',  operator: 'GreaterThan', value: 10000 },
  ],
})

// Dynamic (runs on each request)
collection.addSegment('Renewed this month', async () => {
  const start = new Date()
  start.setDate(1); start.setHours(0, 0, 0, 0)
  return { field: 'renewedAt', operator: 'After', value: start }
})
```

---

## Chart

### Collection-level chart

```typescript
collection.addChart('monthlyRevenue', async (context, resultBuilder) => {
  const rows = await db.query(
    `SELECT date_trunc('month', created_at) AS month, sum(amount) AS total
     FROM orders WHERE status = 'paid' GROUP BY month ORDER BY month`
  )
  return resultBuilder.timeBased('Month', rows.map(r => ({
    date: r.month, value: Number(r.total),
  })))
})
```

### Agent-level chart (global analytics)

```typescript
agent.addChart('totalRevenue', async (_ctx, resultBuilder) => {
  const [{ sum }] = await db.query(`SELECT sum(amount) FROM orders WHERE status = 'paid'`)
  return resultBuilder.value(Number(sum))
})
```

### All result builders

```typescript
resultBuilder.value(n)                          // single number
resultBuilder.value(current, previous)          // with comparison
resultBuilder.distribution({ key: value, ... }) // pie / bar
resultBuilder.timeBased('Month', [              // line chart
  { date: new Date('2024-01-01'), value: 10 },
  { date: new Date('2024-02-01'), value: 15 },
])
resultBuilder.multipleTimeBased('Day', dates, [ // multi-line chart
  { label: 'Sales',   values: [100, 150, 200] },
  { label: 'Returns', values: [10, 15, null] },
])
resultBuilder.percentage(n)                     // 0-100
resultBuilder.objective(current, target)        // progress bar
resultBuilder.leaderboard({ name: score, ... }) // ranked list
resultBuilder.smart(data)                       // custom format
```

Granularities: `'Day'` `'Week'` `'Month'` `'Quarter'` `'Year'`

---

## Search

```typescript
collection.replaceSearch(async (searchString) => ({
  aggregator: 'Or',
  conditions: [
    { field: 'firstName', operator: 'IContains', value: searchString },
    { field: 'lastName',  operator: 'IContains', value: searchString },
    { field: 'email',     operator: 'IContains', value: searchString },
  ],
}))

collection.disableSearch()
```

---

## Filter operators

```typescript
// In-memory emulation — use for computed/virtual fields
collection.emulateFieldFiltering('fullName')            // all operators
collection.emulateFieldOperator('fullName', 'Contains') // one operator

// Custom DB implementation — maps virtual operator to real fields
collection.replaceFieldOperator('fullName', 'Contains', (value) => ({
  aggregator: 'Or',
  conditions: [
    { field: 'firstName', operator: 'Contains', value },
    { field: 'lastName',  operator: 'Contains', value },
  ],
}))
```

---

## Sorting

```typescript
collection.emulateFieldSorting('fullName')            // in-memory sort
collection.replaceFieldSorting('fullName', [          // map to real fields
  { field: 'lastName',  ascending: true },
  { field: 'firstName', ascending: true },
])
collection.disableFieldSorting('computedField')
```

---

## Misc

```typescript
collection.disableCount()                         // removes pagination count query
agent.removeCollection('internalLogs')            // hides from UI (usable internally)
agent.use(advancedExportPlugin, { format: 'xlsx' })
collection.use(createFileField, { fieldname: 'avatar', bucket: 'my-bucket' })
```

---

## ConditionTree operators

```typescript
// Leaf
{ field: 'status', operator: 'Equal', value: 'active' }

// Branch
{ aggregator: 'And' | 'Or', conditions: [leaf1, leaf2] }
```

**Comparison:** `Equal` `NotEqual` `GreaterThan` `LessThan` `In` `NotIn` `Present` `Blank`
**String:** `Contains` `NotContains` `IContains` `StartsWith` `EndsWith` `Like` `ILike` `Match`
**Date:** `Before` `After` `Today` `Yesterday` `Past` `Future` `PreviousWeek` `PreviousMonth` `PreviousQuarter` `PreviousYear` `PreviousXDays` (takes a `value`: number of days)
**Array:** `IncludesAll` `IncludesNone`

---

## Caller

```typescript
context.caller.id         // number
context.caller.email      // string
context.caller.firstName  // string
context.caller.lastName   // string
context.caller.team       // string
context.caller.role       // string
context.caller.timezone   // string — e.g. 'Europe/Paris'
context.caller.tags       // Record<string, string>
```
