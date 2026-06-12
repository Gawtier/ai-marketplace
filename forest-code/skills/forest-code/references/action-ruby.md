# Actions — Ruby

## Basic structure

```ruby
collection.add_action('Label shown in UI', {
  scope: 'Single',                      # 'Single', 'Bulk', or 'Global'
  description: 'Shown as tooltip',      # optional
  submit_button_label: 'Custom label',  # optional, default: 'Submit'
  generate_file: false,                 # set true when execute returns a file
  form: [ /* form fields */ ],
  execute: ->(context, result_builder) {
    # ...
    result_builder.success('Done')
  }
})
```

---

## Scopes

| Scope | When available | `context` methods available |
|-------|---------------|----------------------------|
| `'Single'` | One record selected | `get_record`, `get_record_id`, `get_field` |
| `'Bulk'` | One or more records selected | `get_records`, `get_record_ids` |
| `'Global'` | Always visible — no selection needed | none of the above |

```ruby
# Single
execute: ->(context, result_builder) {
  record = context.get_record(['id', 'email', 'status'])
  id     = context.get_record_id
}

# Bulk
execute: ->(context, result_builder) {
  records = context.get_records(['id', 'email'])
  ids     = context.get_record_ids
}
```

---

## Form field types

### String, Number, Boolean, Date, Dateonly, Time

```ruby
{ label: 'Name',       type: 'String',   is_required: true }
{ label: 'Amount',     type: 'Number',   default_value: 0 }
{ label: 'Confirmed',  type: 'Boolean',  default_value: false }
{ label: 'Due date',   type: 'Date' }
{ label: 'Birth date', type: 'Dateonly' }
{ label: 'Start time', type: 'Time' }
```

### Enum / EnumList

```ruby
{
  label: 'Status',
  type: 'Enum',
  enum_values: ['pending', 'approved', 'rejected'],
  is_required: true
}
{
  label: 'Tags',
  type: 'EnumList',
  enum_values: ['urgent', 'billing', 'technical']
}
```

### File / FileList

```ruby
{ label: 'Attachment', type: 'File' }
{ label: 'Documents',  type: 'FileList' }
```

### Json

```ruby
{ label: 'Metadata', type: 'Json' }
```

### Collection — record picker

```ruby
{
  label: 'Assign to',
  type: 'Collection',
  collection_name: 'User'
}
```

### StringList / NumberList

```ruby
{ label: 'Recipients', type: 'StringList' }
{ label: 'Item IDs',   type: 'NumberList' }
```

---

## Field properties

| Property | Description |
|----------|-------------|
| `type` | `String` `Number` `Boolean` `Date` `Dateonly` `Time` `Enum` `EnumList` `Json` `File` `FileList` `Collection` `StringList` `NumberList` |
| `label` | Displayed to the operator (and the key in `context.form_values` unless `id` is set) |
| `id` | Optional internal identifier; access via `context.form_values['id']` |
| `description` | Help text shown below the field |
| `is_required` | Boolean or `->(context) { boolean }` |
| `is_read_only` | Boolean or `->(context) { boolean }` |
| `default_value` | Value or `->(context) { value }` |
| `value` | `->(context) { value }` — for read-only computed fields |
| `if` | `->(context) { boolean }` — conditional field visibility |
| `enum_values` | Required for `Enum` / `EnumList` (array or `->(context) { array }`) |
| `collection_name` | For `Collection` fields — string or `->(context) { string }` |
| `widget` | UI widget (see below) |
| `placeholder` | Placeholder text |
| `options` | Widget-specific options (e.g. `{ currency: 'USD' }`) |

---

## Widgets

Set `widget` on the field to customize its UI.

```ruby
{ type: 'String',     label: 'Notes',       widget: 'TextArea' }        # multi-line
{ type: 'String',     label: 'Name',        widget: 'TextInput' }       # default for String
{ type: 'StringList', label: 'Tags',        widget: 'TextInputList' }   # list of strings
{ type: 'String',     label: 'Address',     widget: 'AddressAutocomplete', placeholder: 'Type the address' }
{ type: 'Boolean',    label: 'Notify',      widget: 'Checkbox' }
{ type: 'String',     label: 'Priority',    widget: 'Dropdown', options: ['Low', 'Medium', 'High'] }
{ type: 'Enum',       label: 'Plan',        widget: 'RadioGroup',    enum_values: ['Free', 'Pro'] }
{ type: 'EnumList',   label: 'Features',    widget: 'CheckboxGroup', enum_values: ['API', 'SSO'] }
{ type: 'Date',       label: 'Appointment', widget: 'DatePicker' }
{ type: 'Time',       label: 'Opening',     widget: 'TimePicker' }
{ type: 'String',     label: 'Brand color', widget: 'ColorPicker' }
{ type: 'File',       label: 'Picture',     widget: 'FilePicker' }
{ type: 'Json',       label: 'Config',      widget: 'JsonEditor' }
{ type: 'String',     label: 'Assigned to', widget: 'UserDropdown' }    # Forest users — value is the user id
{ type: 'Number',     label: 'Price',       widget: 'CurrencyInput', options: { currency: 'USD' } }
{ type: 'String',     label: 'Content',     widget: 'RichText' }
```

---

## Dynamic form fields

### Dynamic enum_values

```ruby
{
  label: 'Department',
  type: 'Enum',
  enum_values: ->(context) {
    company_id = context.form_values['Company']
    Department.where(company_id: company_id).pluck(:name)
  }
}
```

### Dynamic default_value

```ruby
{
  label: 'Note',
  type: 'String',
  default_value: ->(context) {
    record = context.get_record(['notes'])
    record['notes'] || ''
  }
}
```

### Read-only prefilled from record

```ruby
{
  label: 'Current balance',
  type: 'Number',
  is_read_only: true,
  value: ->(context) {
    record = context.get_record(['balance'])
    record['balance']
  }
}
```

### Conditional display

```ruby
form: [
  {
    label: 'Notification type',
    type: 'Enum',
    enum_values: ['email', 'sms', 'push']
  },
  {
    label: 'Email address',
    type: 'String',
    if: ->(context) { context.form_values['Notification type'] == 'email' },
    is_required: true
  },
  {
    label: 'Phone number',
    type: 'String',
    if: ->(context) { context.form_values['Notification type'] == 'sms' }
  }
]
```

### Dynamic is_required

```ruby
{
  label: 'Justification',
  type: 'String',
  is_required: ->(context) { context.form_values['Amount'] > 1000 }
}
```

### Dynamic collection_name

```ruby
{
  label: 'Entity',
  type: 'Collection',
  collection_name: ->(context) { context.form_values['entityType'] }
}
```

### React to a changed field

```ruby
{
  label: 'Discount',
  type: 'Number',
  is_required: ->(context) { context.has_field_changed('Amount') }
}
```

---

## Layout components

Organize fields with layout elements: `type: 'Layout'` + a `component`. Use `if_condition:` for conditional display (note: **`if_condition`** on layout elements, vs **`if`** on regular fields).

```ruby
# Horizontal separator
{ type: 'Layout', component: 'Separator' }

# Arbitrary HTML
{
  type: 'Layout',
  component: 'HtmlBlock',
  content: ->(context) { "<strong>Hi #{context.form_values['firstName']}</strong>" }
}

# Two fields side by side — exactly two, no nested layouts
{
  type: 'Layout',
  component: 'Row',
  fields: [
    { label: 'gender',  type: 'Enum',   enum_values: ['M', 'F', 'other'] },
    { label: 'specify', type: 'String', if_condition: ->(context) { context.form_values['gender'] == 'other' } }
  ]
}
```

### Multi-page form

When using `Page`, the root form must contain **only** `Page` components.

```ruby
form: [
  {
    type: 'Layout',
    component: 'Page',
    next_button_label: 'Go to address',
    elements: [
      { type: 'String', id: 'firstName', label: 'First name' },
      { type: 'Layout', component: 'Separator' },
      { type: 'Date',   id: 'birthdate', label: 'Birth date' }
    ]
  },
  {
    type: 'Layout',
    component: 'Page',
    previous_button_label: 'Back',
    elements: [
      { type: 'String', id: 'city',    label: 'City' },
      { type: 'String', id: 'country', label: 'Country' }
    ]
  }
]
```

---

## ResultBuilder — full reference

```ruby
# Success
result_builder.success
result_builder.success('Message shown to operator')
result_builder.success('Message', {
  html: '<b>Custom HTML</b>',            # replaces message with HTML
  invalidated: ['Order', 'Customer']     # forces UI refresh of these collections
})

# Error
result_builder.error('Something went wrong')
result_builder.error('Error', { html: '<p>Details</p>' })

# File download — requires generate_file: true on the action definition
result_builder.file(io_stream, 'export.csv', 'text/csv')
result_builder.file(pdf_data, 'report.pdf', 'application/pdf')

# Redirect
result_builder.redirect_to('/path/in/app')

# Webhook
result_builder.webhook(
  'https://hooks.example.com/notify',
  'POST',
  { 'Content-Type' => 'application/json' },
  { order_id: 42, event: 'refunded' }
)

# Set response header
result_builder.set_header('X-Custom', 'value')
```

---

## File generation example

```ruby
collection.add_action('Export to CSV', {
  scope: 'Bulk',
  generate_file: true,
  execute: ->(context, result_builder) {
    records = context.get_records(['id', 'name', 'email', 'created_at'])

    csv = CSV.generate do |csv|
      csv << ['ID', 'Name', 'Email', 'Created At']
      records.each { |r| csv << [r['id'], r['name'], r['email'], r['created_at']] }
    end

    result_builder.file(StringIO.new(csv), 'export.csv', 'text/csv')
  }
})
```

---

## Error handling pattern

```ruby
execute: ->(context, result_builder) {
  begin
    record = context.get_record(['id', 'status'])

    if record['status'] != 'pending'
      return result_builder.error("Cannot process: record is already #{record['status']}")
    end

    OrderProcessor.call(record['id'])
    result_builder.success('Processed', { invalidated: ['Order'] })
  rescue => e
    result_builder.error("Unexpected error: #{e.message}")
  end
}
```

---

## Accessing form values

Form values are keyed by `label` (exact string, case-sensitive):

```ruby
values    = context.form_values
reason    = values['Reason']           # string
amount    = values['Amount']           # number
is_urgent = values['Urgent']           # boolean
due_date  = values['Due date']         # Time object
tags      = values['Tags']             # array of strings
```
