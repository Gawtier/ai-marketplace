# Smart actions — forest-rails (legacy)

Declared in `lib/forest_liana/collections/<model>.rb`; executed by a Rails controller wired in `config/routes.rb`.

## Declaration

```ruby
class Forest::Company
  include ForestLiana::Collection
  collection :Company

  action 'Mark as Live', type: 'single'   # 'bulk' (default) | 'single' | 'global'
end
```

| Option | Notes |
|--------|-------|
| `type` | `'bulk'` (default), `'single'`, `'global'` |
| `fields` | form fields |
| `download` | `true` → file download |
| `endpoint` | default `/forest/actions/{dasherized-name}` |
| `http_method` | default `'POST'` |

## Form fields

```ruby
action 'Upload legal docs', type: 'single', fields: [{
  field: 'Certificate',
  type: 'File',
  is_required: true,
  description: 'The legal document'
}, {
  field: 'amount',
  type: 'Number',           # Boolean, Date, Dateonly, Enum, File, Number, String, ['String']
  default_value: 0
}, {
  field: 'country',
  type: 'Enum',
  enums: ['US', 'FR', 'DE']
}, {
  field: 'company_name',
  reference: 'Company.id'   # record picker
}]
```

Field options: `field`, `type`, `is_required`, `default_value`, `reference`, `enums`, `description`, `widget`, `hook`, `is_read_only`.

## Dynamic forms — hooks

Hooks receive a context hash with `:fields`, `:params`, `:record`. Return `context[:fields]`.

```ruby
action 'Charge credit card', type: 'single',
  fields: [
    { field: 'amount', type: 'Number', is_required: true },
    { field: 'stripe_id', type: 'String' }
  ],
  hooks: {
    load: -> (context) {
      id = context[:params][:data][:attributes][:ids][0]
      customer = Customer.find(id)
      context[:fields].find { |f| f[:field] == 'stripe_id' }[:value] = customer.stripe_id
      context[:fields]
    },
    change: {
      'on_city_change' => -> (context) {
        changed = context[:params][:data][:attributes][:changed_field][:value]
        context[:fields].find { |f| f[:field] == 'zip' }[:value] = zip_for(changed)
        context[:fields]
      }
    }
  }
```

## Controller + routing

```ruby
# app/controllers/forest/companies_controller.rb
class Forest::CompaniesController < ForestLiana::SmartActionsController
  def mark_as_live
    ids = ForestLiana::ResourcesGetter.get_ids_from_request(params, forest_user)
    Company.where(id: ids).update_all(status: 'live')
    head :no_content
  end

  def charge_credit_card
    id     = ForestLiana::ResourcesGetter.get_ids_from_request(params, forest_user).first
    amount = params.dig('data', 'attributes', 'values', 'amount').to_i
    # ... business logic ...
    render json: { success: 'Charge successful!' }
  end
end
```

```ruby
# config/routes.rb — MUST come BEFORE the engine mount
Rails.application.routes.draw do
  namespace :forest do
    post '/actions/mark-as-live'      => 'companies#mark_as_live'
    post '/actions/charge-credit-card' => 'companies#charge_credit_card'
  end
  mount ForestLiana::Engine => '/forest'
end
```

## Reading request data (in the controller)

```ruby
ids    = ForestLiana::ResourcesGetter.get_ids_from_request(params, forest_user)  # handles "select all"
values = params.dig('data', 'attributes', 'values')
intent = params.dig('data', 'attributes', 'action_intent_params')
```

## Responses

| Effect | Code |
|--------|------|
| Default success | `head :no_content` |
| Success toast | `render json: { success: 'msg' }` |
| Error | `render status: 400, json: { error: 'msg' }` |
| HTML | `render json: { html: '<p>…</p>' }` |
| Refresh related | `render json: { success: 'msg', refresh: { relationships: ['rel'] } }` |
| Redirect | `render json: { success: 'msg', redirectTo: 'url-or-internal-path' }` |
| Webhook | `render json: { webhook: { url:, method:, headers:, body: } }` |
| File (`download: true`) | `send_data data, filename:, type:, disposition: 'attachment'` |
