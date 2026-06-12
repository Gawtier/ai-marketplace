# Custom datasource — Ruby

## Structure

```ruby
require 'forest_admin_datasource_toolkit'

class MyDatasource < ForestAdminDatasourceToolkit::Datasource
  def initialize
    super
    add_collection(MyCollection.new(self))
  end
end

class MyCollection < ForestAdminDatasourceToolkit::Collection
  def initialize(datasource)
    super('MyCollection', datasource)

    add_field('id', ForestAdminDatasourceToolkit::Schema::ColumnSchema.new(
      column_type: 'Number',
      is_primary_key: true,
      filter_operators: Set['Equal', 'In'],
      is_sortable: true
    ))

    add_field('name', ForestAdminDatasourceToolkit::Schema::ColumnSchema.new(
      column_type: 'String',
      is_read_only: false,
      filter_operators: Set['Equal', 'Contains', 'IContains', 'StartsWith'],
      is_sortable: true
    ))

    add_field('status', ForestAdminDatasourceToolkit::Schema::ColumnSchema.new(
      column_type: 'Enum',
      enum_values: ['active', 'inactive'],
      filter_operators: Set['Equal', 'In']
    ))
  end

  # Implement all 5 CRUD methods below
end
```

---

## Field definition

```ruby
add_field('field_name', ForestAdminDatasourceToolkit::Schema::ColumnSchema.new(
  column_type: 'String',   # see column types below
  is_primary_key: false,
  is_read_only: false,
  is_sortable: true,
  filter_operators: Set['Equal', 'Contains'],
  # for Enum:
  enum_values: ['a', 'b']
))
```

**Column types:** `Boolean` `Date` `Dateonly` `Enum` `Json` `Number` `Point` `String` `Timeonly` `Uuid`

**Composite types:**
```ruby
column_type: { 'first_name' => 'String', 'last_name' => 'String' }  # nested object
column_type: ['String']                                               # array of primitives
column_type: [{ 'content' => 'String', 'date' => 'Date' }]           # array of objects
```

---

## Relationships

```ruby
# Many-to-one: this collection has a FK pointing to another collection
add_field('author', ForestAdminDatasourceToolkit::Schema::Relations::ManyToOneSchema.new(
  foreign_key: 'author_id',       # FK on this collection
  foreign_key_target: 'id',       # PK on target collection
  foreign_collection: 'Author'
))

# One-to-many: another collection has a FK pointing here
add_field('comments', ForestAdminDatasourceToolkit::Schema::Relations::OneToManySchema.new(
  origin_key: 'post_id',          # FK on the other collection
  origin_key_target: 'id',        # PK on this collection
  foreign_collection: 'Comment'
))

# Many-to-many (junction table)
add_field('tags', ForestAdminDatasourceToolkit::Schema::Relations::ManyToManySchema.new(
  origin_key: 'post_id',
  origin_key_target: 'id',
  foreign_key: 'tag_id',
  foreign_key_target: 'id',
  through_collection: 'PostTag',
  foreign_collection: 'Tag'
))
```

---

## list()

```ruby
def list(caller, filter, projection)
  # filter.condition_tree  — nested condition tree (may be nil)
  # filter.page            — { skip:, limit: }
  # filter.sort            — [{ field:, ascending: }]
  # projection             — array of field names to return

  MyApi.list(
    where:    translate_condition_tree(filter.condition_tree),
    skip:     filter.page&.skip || 0,
    limit:    filter.page&.limit || 20,
    order_by: filter.sort&.map { |s| { s[:field] => s[:ascending] ? :asc : :desc } },
    select:   projection
  )
end
```

---

## create()

```ruby
def create(caller, data)
  data.map { |record| MyApi.create(record) }
  # must return the created records with generated IDs
end
```

---

## update()

```ruby
def update(caller, filter, data)
  records = list(caller, filter, ['id'])
  ids = records.map { |r| r['id'] }
  MyApi.update_many(ids, data)
  # no return value
end
```

---

## delete()

```ruby
def delete(caller, filter)
  records = list(caller, filter, ['id'])
  ids = records.map { |r| r['id'] }
  MyApi.delete_many(ids)
  # no return value
end
```

---

## aggregate()

```ruby
def aggregate(caller, filter, aggregation, limit = nil)
  # aggregation.operation  — 'Count' | 'Sum' | 'Avg' | 'Max' | 'Min'
  # aggregation.field      — field to aggregate on (nil for Count)
  # aggregation.groups     — [{ field: }] — group-by fields

  # Simple count example
  if aggregation.operation == 'Count' && aggregation.groups.empty?
    count = MyApi.count(translate_condition_tree(filter.condition_tree))
    return [{ 'value' => count, 'group' => {} }]
  end

  rows = MyApi.aggregate(
    operation: aggregation.operation,
    field:     aggregation.field,
    group_by:  aggregation.groups.map { |g| g[:field] },
    where:     translate_condition_tree(filter.condition_tree)
  )

  rows.map { |r| { 'value' => r['value'], 'group' => r['group'] } }
end
```

---

## Registering the datasource

```ruby
ForestAdmin.agent.add_datasource(MyDatasource.new)
```

---

## ConditionTree translation helper (example)

```ruby
def translate_condition_tree(tree)
  return {} if tree.nil?

  if tree.is_a?(ForestAdminDatasourceToolkit::Components::Query::ConditionTree::Nodes::ConditionTreeLeaf)
    translate_leaf(tree.field, tree.operator, tree.value)
  else
    sub = tree.conditions.map { |c| translate_condition_tree(c) }
    tree.aggregator == 'And' ? { and: sub } : { or: sub }
  end
end
```
