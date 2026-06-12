# Smart collections, charts & routes — forest-rails (legacy)

## Smart collection

```ruby
# lib/forest_liana/collections/customer_stat.rb
class Forest::CustomerStat
  include ForestLiana::Collection
  collection :CustomerStat, is_searchable: true

  field :id,           type: 'Number', is_read_only: true   # mandatory unique id
  field :email,        type: 'String', is_read_only: true
  field :orders_count, type: 'Number', is_read_only: true
  field :total_amount, type: 'Number', is_read_only: true
end
```

Provide the records via a controller `index`; wire the route **before** the engine mount.

```ruby
# config/routes.rb
namespace :forest do
  get '/CustomerStat' => 'customer_stats#index'
end
mount ForestLiana::Engine => '/forest'
```

## Serializing records — JSON:API

```ruby
class BaseSerializer
  include JSONAPI::Serializer
  def type; 'customerStat'; end
  def format_name(name);   name.to_s.underscore; end
  def unformat_name(name); name.to_s.dasherize;  end
end

class CustomerStatSerializer < BaseSerializer
  attribute :email
  attribute :orders_count
  attribute :total_amount
end

# in the controller
def index
  limit  = params[:page][:size].to_i
  offset = (params[:page][:number].to_i - 1) * limit
  rows   = Customer.find_by_sql(["... LIMIT ? OFFSET ?", limit, offset])
  count  = Customer.count
  render json: CustomerStatSerializer.serialize(rows, is_collection: true, meta: { count: count })
end
```

Implement search yourself from `params[:search]` (sanitize before interpolating into SQL).

## API-based charts

Return a `ForestLiana::Model::Stat` from a controller; wire `post '/stats/<name>'` before the mount.

```ruby
class Forest::ChartsController < ForestLiana::ApplicationController
  def mrr            # value
    render json: serialize_model(ForestLiana::Model::Stat.new(value: 4200))
  end

  def by_country     # repartition (pie)
    render json: serialize_model(ForestLiana::Model::Stat.new(value: [{ key: 'US', value: 42 }]))
  end

  def charges_per_day # time-based (line)
    render json: serialize_model(ForestLiana::Model::Stat.new(
      value: [{ label: '01/03/2018', values: { value: 10 } }]))
  end

  def goal           # objective
    render json: serialize_model(ForestLiana::Model::Stat.new(value: { value: 10, objective: 678 }))
  end
end
```

> A **Smart Chart** is a frontend component; only its backend data route lives here.

## Default routes

`POST /<c>`, `PUT/DELETE /<c>/:id`, `GET /<c>`, `/<c>/count`, `/<c>/:id`, `/<c>.csv`, plus relationship routes `GET|POST|DELETE /<c>/:id/relationships/<rel>`.

## Extend a route — keep default behavior

Re-open the generated controller and alias the default action.

```ruby
if ForestLiana::UserSpace.const_defined?('CompanyController')
  ForestLiana::UserSpace::CompanyController.class_eval do
    alias_method :default_index, :index
    def index
      params['searchExtended'] = '1'   # your tweak
      default_index                    # → Forest default
    end
  end
end
```

## Override a route — replace default behavior

```ruby
if ForestLiana::UserSpace.const_defined?('CompanyController')
  ForestLiana::UserSpace::CompanyController.class_eval do
    alias_method :default_destroy, :destroy
    def destroy
      if params['id'] == '50'
        render status: 403, plain: 'This record is protected.'
      else
        default_destroy
      end
    end
  end
end
```
