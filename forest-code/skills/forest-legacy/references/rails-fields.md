# Smart fields, segments & relationships — forest-rails (legacy)

Declared inside a `ForestLiana::Collection` block. `object` is the current record.

## Smart field

```ruby
class Forest::Customer
  include ForestLiana::Collection
  collection :Customer

  field :fullname, type: 'String' do
    "#{object.firstname} #{object.lastname}"
  end
end
```

Types: `Boolean`, `Date`, `Dateonly`, `Enum`, `File`, `Number`, `Json`, `String`, `['String']`.

### Writable — `set`

```ruby
set_fullname = lambda do |user_params, fullname|
  first, last = fullname.split
  user_params[:firstname] = first
  user_params[:lastname]  = last
  user_params                      # must return user_params
end

field :fullname, type: 'String', set: set_fullname do
  "#{object.firstname} #{object.lastname}"
end
```

### Searchable — `search`

```ruby
search_fullname = lambda do |query, search|
  first, last = search.split
  query.where_clause.send(:predicates)[0] << " OR (firstname = '#{first}' AND lastname = '#{last}')"
  query
end

field :fullname, type: 'String', search: search_fullname do
  "#{object.firstname} #{object.lastname}"
end
```

### Filterable — `filter`

```ruby
filter_fullname = lambda do |condition, where|
  first = condition['value']&.split&.first
  case condition['operator']
  when 'equal'     then "firstname = '#{first}'"
  when 'ends_with' then "firstname LIKE '%#{first}'"
  end
end

field :fullname, type: 'String', is_filterable: true, filter: filter_fullname do
  "#{object.firstname} #{object.lastname}"
end
```

Options: `type`, `enums`, `description`, `reference`, `is_read_only` (default `true` unless `set`), `is_required`, `is_filterable`, `set`, `search`, `filter`.

## Smart segment

```ruby
segment 'Bestsellers' do
  ids = Product.joins(:orders).group('products.id').order('count(orders.id) DESC').limit(10).pluck('products.id')
  { id: ids }
end
```

## Smart relationship

### belongs_to

```ruby
search_delivery_address = lambda do |query, search|
  query.joins(customer: :address).where("addresses.country ILIKE ?", "%#{search}%")
end

belongs_to :delivery_address, reference: 'Address.id', search: search_delivery_address do
  object.customer.address
end
```

### has_many — declare, then resolve via controller route

```ruby
# collection
has_many :buyers, type: ['String'], reference: 'Customer.id'
```

```ruby
# config/routes.rb (before engine mount)
namespace :forest do
  get '/Product/:product_id/buyers' => 'products#buyers'
end
```

```ruby
# app/controllers/forest/products_controller.rb
class Forest::ProductsController < ForestLiana::ApplicationController
  def buyers
    limit  = params['page']['size'].to_i
    offset = (params['page']['number'].to_i - 1) * limit
    product = Product.find(params['product_id'])
    customers = Customer.where(order_id: product.orders.ids)
    render json: serialize_models(customers.limit(limit).offset(offset), meta: { count: customers.count })
  end
end
```
