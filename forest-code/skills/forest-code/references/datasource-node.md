# Custom datasource — Node.js

## Structure

```typescript
import { BaseDataSource, BaseCollection } from '@forestadmin/datasource-toolkit'

class MyDataSource extends BaseDataSource {
  constructor() {
    super()
    this.addCollection(new MyCollection(this))
  }
}

class MyCollection extends BaseCollection {
  constructor(dataSource: BaseDataSource) {
    super('MyCollection', dataSource)

    this.addField('id', {
      type: 'Column',
      columnType: 'Number',
      isPrimaryKey: true,
      filterOperators: new Set(['Equal', 'In']),
      isSortable: true,
    })

    this.addField('name', {
      type: 'Column',
      columnType: 'String',
      isReadOnly: false,
      filterOperators: new Set(['Equal', 'Contains', 'IContains', 'StartsWith']),
      isSortable: true,
    })

    this.addField('status', {
      type: 'Column',
      columnType: 'Enum',
      enumValues: ['active', 'inactive'],
      filterOperators: new Set(['Equal', 'In']),
    })
  }

  // Implement all 5 CRUD methods below
}
```

---

## Field definition

```typescript
this.addField('fieldName', {
  type: 'Column',          // 'Column' | 'ManyToOne' | 'OneToMany' | 'ManyToMany' | 'OneToOne'
  columnType: 'String',    // see column types below
  isPrimaryKey: false,
  isReadOnly: false,
  unique: false,
  isSortable: true,
  filterOperators: new Set(['Equal', 'Contains']),
  // for Enum:
  enumValues: ['a', 'b'],
})
```

**Column types:** `Boolean` `Date` `Dateonly` `Enum` `Json` `Number` `Point` `String` `Timeonly` `Uuid`

**Composite types:**
```typescript
columnType: { firstName: 'String', lastName: 'String' }  // nested object
columnType: ['String']                                    // array of primitives
columnType: [{ content: 'String', date: 'Date' }]        // array of objects
```

---

## Relationships

```typescript
// Many-to-one: this collection has a FK pointing to another collection
this.addField('author', {
  type: 'ManyToOne',
  foreignKey: 'author_id',       // FK field on this collection
  foreignKeyTarget: 'id',        // PK field on target collection
  foreignCollection: 'Author',
})

// One-to-many: another collection has a FK pointing here
this.addField('comments', {
  type: 'OneToMany',
  originKey: 'post_id',          // FK field on the other collection
  originKeyTarget: 'id',         // PK field on this collection
  foreignCollection: 'Comment',
})

// Many-to-many (junction table)
this.addField('tags', {
  type: 'ManyToMany',
  originKey: 'post_id',
  originKeyTarget: 'id',
  foreignKey: 'tag_id',
  foreignKeyTarget: 'id',
  throughCollection: 'PostTag',
  foreignCollection: 'Tag',
})
```

---

## list()

```typescript
async list(caller: Caller, filter: PaginatedFilter, projection: Projection): Promise<RecordData[]> {
  // filter.conditionTree   — nested condition tree (may be undefined)
  // filter.page            — { skip: number, limit: number }
  // filter.sort            — [{ field: string, ascending: boolean }]
  // projection             — array of field names to return

  const records = await MyApi.list({
    where: translateConditionTree(filter.conditionTree),
    skip: filter.page?.skip ?? 0,
    limit: filter.page?.limit ?? 20,
    orderBy: filter.sort?.map(s => ({ [s.field]: s.ascending ? 'asc' : 'desc' })),
    select: projection,
  })

  return records
}
```

---

## create()

```typescript
async create(caller: Caller, records: RecordData[]): Promise<RecordData[]> {
  const created = await Promise.all(records.map(r => MyApi.create(r)))
  return created  // must return the created records with generated IDs
}
```

---

## update()

```typescript
async update(caller: Caller, filter: Filter, patch: RecordData): Promise<void> {
  const records = await this.list(caller, filter, ['id'])
  const ids = records.map(r => r['id'])
  await MyApi.updateMany(ids, patch)
  // no return value
}
```

---

## delete()

```typescript
async delete(caller: Caller, filter: Filter): Promise<void> {
  const records = await this.list(caller, filter, ['id'])
  const ids = records.map(r => r['id'])
  await MyApi.deleteMany(ids)
  // no return value
}
```

---

## aggregate()

```typescript
async aggregate(
  caller: Caller,
  filter: Filter,
  aggregation: Aggregation,
  limit?: number,
): Promise<AggregateResult[]> {
  // aggregation.operation  — 'Count' | 'Sum' | 'Avg' | 'Max' | 'Min'
  // aggregation.field      — field to aggregate on (undefined for Count)
  // aggregation.groups     — [{ field: string }] — group-by fields

  // Simple count example
  if (aggregation.operation === 'Count' && !aggregation.groups?.length) {
    const count = await MyApi.count(translateConditionTree(filter.conditionTree))
    return [{ value: count, group: {} }]
  }

  // Grouped example
  const rows = await MyApi.aggregate({
    operation: aggregation.operation,
    field: aggregation.field,
    groupBy: aggregation.groups?.map(g => g.field),
    where: translateConditionTree(filter.conditionTree),
  })

  return rows.map(r => ({ value: r.value, group: r.group }))
}
```

---

## Registering the datasource

```typescript
import { createAgent } from '@forestadmin/agent'

const agent = createAgent({ ... })
agent.addDataSource(new MyDataSource())
await agent.start()
```

---

## ConditionTree translation helper (example)

```typescript
function translateConditionTree(tree: ConditionTree | undefined): WhereClause {
  if (!tree) return {}
  if (tree.type === 'Leaf') {
    return translateLeaf(tree.field, tree.operator, tree.value)
  }
  // Branch
  const sub = tree.conditions.map(translateConditionTree)
  return tree.aggregator === 'And' ? { AND: sub } : { OR: sub }
}
```

---

## Replication datasource (simpler alternative)

For read-heavy APIs with limited query capabilities, use `@forestadmin/datasource-replica`:

```typescript
import { createReplicaDataSource } from '@forestadmin/datasource-replica'

agent.addDataSource(
  createReplicaDataSource({
    schema: [
      {
        name: 'products',
        fields: { id: 'Number', name: 'String', price: 'Number' },
        primaryKey: 'id',
      },
    ],
    pullDumpHandler: async ({ previousDumpAt }) => {
      const items = await MyApi.getAll({ since: previousDumpAt })
      return {
        more: false,
        entries: items.map(i => ({ collection: 'products', record: i })),
      }
    },
  })
)
```
