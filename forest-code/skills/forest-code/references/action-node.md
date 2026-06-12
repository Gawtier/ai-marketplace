# Actions — Node.js

## Basic structure

```typescript
collection.addAction('Label shown in UI', {
  scope: 'Single' | 'Bulk' | 'Global',
  description: 'Shown as tooltip',       // optional
  submitButtonLabel: 'Custom label',      // optional, default: 'Submit'
  generateFile: false,                    // set true when execute returns a file
  form: [ /* FormElement[] */ ],
  execute: async (context, resultBuilder) => {
    // ...
    return resultBuilder.success('Done')
  },
})
```

---

## Scopes

| Scope | When available | `context` methods available |
|-------|---------------|----------------------------|
| `Single` | One record selected | `getRecord`, `getRecordId`, `getField`, `getCompositeRecordId` |
| `Bulk` | One or more records selected | `getRecords`, `getRecordIds`, `getCompositeRecordIds` |
| `Global` | Always visible — no selection needed | none of the above |

```typescript
// Single
execute: async (context, resultBuilder) => {
  const record = await context.getRecord(['id', 'email', 'status'])
  const id = await context.getRecordId()
}

// Bulk
execute: async (context, resultBuilder) => {
  const records = await context.getRecords(['id', 'email'])
  const ids = await context.getRecordIds()
}
```

---

## Form field types

### String, Number, Boolean, Date, Dateonly, Time

```typescript
{ label: 'Name',       type: 'String',   isRequired: true }
{ label: 'Amount',     type: 'Number',   defaultValue: 0 }
{ label: 'Confirmed',  type: 'Boolean',  defaultValue: false }
{ label: 'Due date',   type: 'Date' }
{ label: 'Birth date', type: 'Dateonly' }
{ label: 'Start time', type: 'Time' }
```

### Enum / EnumList

```typescript
{
  label: 'Status',
  type: 'Enum',
  enumValues: ['pending', 'approved', 'rejected'],
  isRequired: true,
}
{
  label: 'Tags',
  type: 'EnumList',
  enumValues: ['urgent', 'billing', 'technical'],
}
```

### File / FileList

```typescript
{ label: 'Attachment', type: 'File' }
{ label: 'Documents',  type: 'FileList' }
```

### Json

```typescript
{ label: 'Metadata', type: 'Json' }
```

### Collection — record picker

```typescript
{
  label: 'Assign to',
  type: 'Collection',
  collectionName: 'users',
}
```

### StringList / NumberList

```typescript
{ label: 'Recipients', type: 'StringList' }
{ label: 'Item IDs',   type: 'NumberList' }
```

---

## Field properties

Every form field accepts:

| Property | Description |
|----------|-------------|
| `type` | `String` `Number` `Boolean` `Date` `Dateonly` `Time` `Enum` `EnumList` `Json` `File` `FileList` `Collection` `StringList` `NumberList` |
| `label` | Displayed to the operator (and the key in `context.formValues` unless `id` is set) |
| `id` | Optional internal identifier — set it to keep a stable key when the label may change; access via `context.formValues['id']` |
| `description` | Help text shown below the field |
| `isRequired` | Boolean or `(context) => boolean` |
| `isReadOnly` | Boolean or `(context) => boolean` |
| `defaultValue` | Value or `async (context) => value` |
| `value` | `async (context) => value` — for read-only computed fields |
| `if` | `(context) => boolean` — conditional visibility |
| `enumValues` | Required for `Enum` / `EnumList` (array or `(context) => array`) |
| `collectionName` | For `Collection` fields — string or `(context) => string` |
| `widget` | UI widget (see below) |
| `placeholder` | Placeholder text |
| `options` | Widget-specific options (e.g. `{ currency: 'USD' }`) |

---

## Widgets

Widgets customize a field's UI. Set `widget` on the field.

```typescript
{ type: 'String',   label: 'Notes',       widget: 'TextArea' }        // multi-line
{ type: 'String',   label: 'Name',        widget: 'TextInput' }       // default for String
{ type: 'StringList', label: 'Tags',      widget: 'TextInputList' }   // list of strings
{ type: 'String',   label: 'Address',     widget: 'AddressAutocomplete', placeholder: 'Type the address' }
{ type: 'Boolean',  label: 'Notify',      widget: 'Checkbox' }
{ type: 'Enum',     label: 'Plan',        widget: 'RadioGroup',  enumValues: ['Free', 'Pro'] }
{ type: 'EnumList', label: 'Features',    widget: 'CheckboxGroup', enumValues: ['API', 'SSO'] }
{ type: 'Date',     label: 'Appointment', widget: 'DatePicker' }
{ type: 'Time',     label: 'Opening',     widget: 'TimePicker' }
{ type: 'String',   label: 'Brand color', widget: 'ColorPicker' }
{ type: 'File',     label: 'Picture',     widget: 'FilePicker' }
{ type: 'Json',     label: 'Config',      widget: 'JsonEditor' }
{ type: 'String',   label: 'Assigned to', widget: 'UserDropdown' }    // Forest users — value is the user id
{ type: 'Number',   label: 'Price',       widget: 'CurrencyInput', options: { currency: 'USD' } }
{ type: 'String',   label: 'Content',     widget: 'RichText' }
```

### Static dropdown

```typescript
{
  label: 'Country',
  type: 'String',
  widget: 'Dropdown',
  options: [
    { label: 'France',         value: 'FR' },
    { label: 'United Kingdom', value: 'UK' },
    { label: 'United States',  value: 'US' },
  ],
  placeholder: 'Select a country',
  search: 'disabled',  // 'disabled' | 'static'
}
```

### Dynamic dropdown (search as you type)

```typescript
{
  label: 'Product',
  type: 'String',
  widget: 'Dropdown',
  search: 'dynamic',
  options: async (context, searchValue) => {
    const products = await searchProducts(searchValue)
    return products.map(p => ({ label: p.name, value: String(p.id) }))
  },
}
```

---

## Dynamic form fields

### Dynamic enumValues

```typescript
{
  label: 'Department',
  type: 'Enum',
  enumValues: async (context) => {
    const companyId = context.formValues['Company']
    return getDepartments(companyId)
  },
}
```

### Dynamic defaultValue

```typescript
{
  label: 'Note',
  type: 'String',
  defaultValue: async (context) => {
    const record = await context.getRecord(['notes'])
    return record.notes ?? ''
  },
}
```

### Read-only prefilled from record

```typescript
{
  label: 'Current balance',
  type: 'Number',
  isReadOnly: true,
  value: async (context) => {
    const record = await context.getRecord(['balance'])
    return record.balance
  },
}
```

### Conditional display

```typescript
form: [
  {
    label: 'Notification type',
    type: 'Enum',
    enumValues: ['email', 'sms', 'push'],
  },
  {
    label: 'Email address',
    type: 'String',
    if: (context) => context.formValues['Notification type'] === 'email',
    isRequired: true,
  },
  {
    label: 'Phone number',
    type: 'String',
    if: (context) => context.formValues['Notification type'] === 'sms',
  },
]
```

### Detect form field changes (re-validation on change)

```typescript
{
  label: 'Discount',
  type: 'Number',
  isRequired: async (context) => context.hasFieldChanged('Amount'),
}
```

---

## Layout components

Organize fields with layout elements. A layout element has `type: 'Layout'` and a `component`. Supports `if: (context) => boolean` for conditional display.

```typescript
// Horizontal separator
{ type: 'Layout', component: 'Separator' }

// Arbitrary HTML (content is a string or a function)
{
  type: 'Layout',
  component: 'HtmlBlock',
  content: (ctx) => `<strong>Hi ${ctx.formValues.firstName}</strong>`,
}

// Two fields side by side — exactly two, no nested layouts
{
  type: 'Layout',
  component: 'Row',
  fields: [
    { label: 'gender', type: 'Enum', enumValues: ['M', 'F', 'other'] },
    { label: 'specify', type: 'String', if: (ctx) => ctx.formValues?.gender === 'other' },
  ],
}
```

### Multi-page form

When using `Page`, the root form must contain **only** `Page` components (no mixing fields and pages, no nested pages).

```typescript
form: [
  {
    type: 'Layout',
    component: 'Page',
    nextButtonLabel: 'Go to address',
    elements: [
      { type: 'String', id: 'firstName', label: 'First name' },
      { type: 'Layout', component: 'Separator' },
      { type: 'Date', id: 'birthdate', label: 'Birth date' },
    ],
  },
  {
    type: 'Layout',
    component: 'Page',
    previousButtonLabel: 'Back',
    elements: [
      { type: 'String', id: 'city', label: 'City' },
      { type: 'String', id: 'country', label: 'Country' },
    ],
  },
]
```

---

## ResultBuilder — full reference

```typescript
// Success
resultBuilder.success()
resultBuilder.success('Message shown to operator')
resultBuilder.success('Message', {
  html: '<b>Custom HTML</b>',          // replaces the message with HTML
  invalidated: ['orders', 'customers'] // forces UI refresh of these collections
})

// Error
resultBuilder.error('Something went wrong')
resultBuilder.error('Error', { html: '<p>Details</p>' })

// File download — requires generateFile: true on the action definition
resultBuilder.file(readableStream, 'export.csv', 'text/csv')
resultBuilder.file(buffer, 'report.pdf', 'application/pdf')

// Redirect
resultBuilder.redirectTo('/path/in/app')

// Webhook
resultBuilder.webhook('https://hooks.example.com/notify', 'POST', {
  'Content-Type': 'application/json',
}, { orderId: 42, event: 'refunded' })

// Set response header
resultBuilder.setHeader('X-Custom', 'value')
```

---

## File generation example

```typescript
collection.addAction('Export to CSV', {
  scope: 'Bulk',
  generateFile: true,
  execute: async (context, resultBuilder) => {
    const records = await context.getRecords(['id', 'name', 'email', 'createdAt'])

    const csv = [
      'ID,Name,Email,Created At',
      ...records.map(r => `${r.id},"${r.name}",${r.email},${r.createdAt}`),
    ].join('\n')

    const stream = Readable.from([csv])
    return resultBuilder.file(stream, 'export.csv', 'text/csv')
  },
})
```

---

## Error handling patterns

```typescript
execute: async (context, resultBuilder) => {
  try {
    const record = await context.getRecord(['id', 'status'])

    if (record.status !== 'pending') {
      return resultBuilder.error(`Cannot process: record is already ${record.status}`)
    }

    await processRecord(record.id)
    return resultBuilder.success('Processed', { invalidated: ['orders'] })
  } catch (err) {
    return resultBuilder.error(`Unexpected error: ${err.message}`)
  }
}
```

---

## Accessing form values

Form values are keyed by `label` (exact string, case-sensitive):

```typescript
const { formValues } = context
const reason   = formValues['Reason']          // string
const amount   = formValues['Amount']          // number
const isUrgent = formValues['Urgent']          // boolean
const dueDate  = formValues['Due date']        // Date object
const tags     = formValues['Tags']            // string[]
```
