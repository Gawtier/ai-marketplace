# Forest Admin — Ruby general reference

Segments, charts, search, filtering, sorting, and misc. For actions see `action-ruby.md`, for hooks/overrides see `hook-ruby.md`, for fields/relationships see `field-ruby.md`.

---

## Segment

```ruby
# Static
collection.add_segment('VIP customers', {
  aggregator: 'And',
  conditions: [
    { field: 'status',          operator: 'Equal',       value: 'active' },
    { field: 'lifetime_value',  operator: 'GreaterThan', value: 10_000 }
  ]
})

# Dynamic (runs on each request)
collection.add_segment('Renewed this month', ->(context) {
  { field: 'renewed_at', operator: 'After', value: Date.today.beginning_of_month }
})
```

---

## Chart

### Collection-level chart

```ruby
collection.add_chart('monthly_revenue', ->(context, result_builder) {
  data = Order.paid.group_by_month(:created_at).sum(:amount)
  result_builder.time_based('Month', data)
})
```

### Agent-level chart (global analytics)

```ruby
ForestAdmin.agent.add_chart('total_revenue', ->(context, result_builder) {
  result_builder.value(Order.paid.sum(:amount))
})
```

### All result builders

```ruby
result_builder.value(n)                            # single number
result_builder.value(current, previous)            # with comparison
result_builder.distribution({ key: value, ... })   # pie / bar
result_builder.time_based('Month', data)           # line chart — hash or array
result_builder.percentage(n)                       # 0-100
result_builder.objective(current, target)          # progress bar
result_builder.leaderboard({ name: score, ... })   # ranked list
result_builder.smart(data)                         # custom format
```

Granularities: `'Day'` `'Week'` `'Month'` `'Quarter'` `'Year'`

---

## Search

```ruby
collection.replace_search(->(search_string, _is_extended) {
  {
    aggregator: 'Or',
    conditions: [
      { field: 'first_name', operator: 'IContains', value: search_string },
      { field: 'last_name',  operator: 'IContains', value: search_string },
      { field: 'email',      operator: 'IContains', value: search_string }
    ]
  }
})

collection.disable_search
```

---

## Filter operators

```ruby
# In-memory emulation — use for computed/virtual fields
collection.emulate_field_filtering('full_name')
collection.emulate_field_operator('full_name', 'Contains')

# Custom DB implementation
collection.replace_field_operator('full_name', 'Contains', ->(value) {
  {
    aggregator: 'Or',
    conditions: [
      { field: 'first_name', operator: 'Contains', value: value },
      { field: 'last_name',  operator: 'Contains', value: value }
    ]
  }
})
```

---

## Sorting

```ruby
collection.emulate_field_sorting('full_name')
collection.replace_field_sorting('full_name', [
  { field: 'last_name',  ascending: true },
  { field: 'first_name', ascending: true }
])
collection.disable_field_sorting('computed_field')
```

---

## Misc

```ruby
collection.disable_count
ForestAdmin.agent.remove_collection('InternalLog')
```

---

## ConditionTree operators

```ruby
# Leaf
{ field: 'status', operator: 'Equal', value: 'active' }

# Branch
{ aggregator: 'And', conditions: [leaf1, leaf2] }
```

**Comparison:** `Equal` `NotEqual` `GreaterThan` `LessThan` `In` `NotIn` `Present` `Blank`
**String:** `Contains` `NotContains` `IContains` `StartsWith` `EndsWith` `Like` `ILike` `Match`
**Date:** `Before` `After` `Today` `Yesterday` `Past` `Future` `PreviousWeek` `PreviousMonth` `PreviousQuarter` `PreviousYear` `PreviousXDays` (takes a `value`: number of days)
**Array:** `IncludesAll` `IncludesNone`

---

## Caller

```ruby
context.caller.id          # integer
context.caller.email       # string
context.caller.first_name  # string
context.caller.last_name   # string
context.caller.team        # string
context.caller.role        # string
context.caller.timezone    # string — e.g. 'Europe/Paris'
```
