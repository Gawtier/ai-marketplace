---
name: forest-legacy
description: Use when maintaining an EXISTING legacy Forest Admin agent — forest-express-sequelize, forest-express-mongoose, or forest-rails (forest_liana). Triggers on Smart Actions, Smart Fields, Smart Segments, Smart Collections, Smart Relationships, Smart Charts, or route extend/override in legacy projects (files under /forest, /routes, or lib/forest_liana). NOT for the modern @forestadmin/agent / forest_admin_agent — use forest-code for those.
---

# Forest Admin Legacy Code

Maintain customization code for **legacy** Forest Admin agents. These are a different product generation from the modern agent, with a different paradigm (Smart Actions/Fields/Segments + routes/serializers), **not** the `customizeCollection` / `addAction` decorator API.

## Step 0 — Confirm it is a legacy project

Before generating anything, check the project manifest:

- **`package.json`** contains `forest-express-sequelize` → legacy JS, SQL stack
- **`package.json`** contains `forest-express-mongoose` → legacy JS, Mongo stack
- **`Gemfile`** contains `forest_liana` → legacy Ruby (forest-rails)
- **`package.json`** contains `@forestadmin/agent`, or **`Gemfile`** contains `forest_admin_agent` → **modern**: stop and use the **forest-code** skill instead.

If the manifest is ambiguous, ask the user which package they run before writing code. Legacy and modern APIs are mutually incompatible — emitting one for the other produces non-working code.

## Step 1 — Collect context (one message)

1. **Stack** — `forest-express-sequelize`, `forest-express-mongoose`, or `forest-rails`?
2. **Feature** — what to implement (see index below)?
3. **Context** — collection/model name(s), field names and types, business logic.

No placeholders. Use the exact names the user provides.

## Step 2 — Load reference files, then generate

| Feature | forest-express (sequelize/mongoose) | forest-rails |
|---------|-------------------------------------|--------------|
| Smart action (+ form, dynamic form, action intent) | `references/express-actions.md` | `references/rails-actions.md` |
| Smart field, smart segment, smart relationship | `references/express-fields.md` | `references/rails-fields.md` |
| Smart collection, serialization, charts, route extend/override | `references/express-collections.md` | `references/rails-collections.md` |

For **forest-express**, also load the model-layer file for the project's ORM:

- SQL → `references/sequelize.md` (model definitions + query idioms inside fields/routes)
- Mongo → `references/mongoose.md`

Load only what the feature requires. Write code immediately after reading.

---

## Feature index

| Feature | File group |
|---------|-----------|
| Smart action (single/bulk/global), form fields, dynamic form hooks, action intent | actions |
| Smart field (get/set/search/filter) | fields |
| Smart segment | fields |
| Smart (belongsTo / hasMany) relationship | fields |
| Smart collection + record serialization | collections |
| API-based chart route | collections |
| Extend / override a default route | collections |
| ORM query idioms (Sequelize / Mongoose) | sequelize / mongoose |

> **Smart Views** are frontend (template/components) — out of scope for this backend skill.

---

## Production checklist

- [ ] Correct file location: `/forest/<collection>.js` + `/routes/<collection>.js` (express) or `lib/forest_liana/collections/<model>.rb` + controller + `config/routes.rb` (rails)
- [ ] Custom routes registered **before** `mount ForestLiana::Engine` (rails)
- [ ] Smart action route guarded by `permissionMiddlewareCreator.smartAction()` (express)
- [ ] Selected ids read via `RecordsGetter`/`getIdsFromRequest` (express) or `ForestLiana::ResourcesGetter.get_ids_from_request` (rails) — handles "select all"
- [ ] `set` on a smart field returns the modified record / params
- [ ] Records serialized to JSON:API before sending (`RecordSerializer` / JSONAPI serializer)
- [ ] No `console.log` / `puts` / debug statements; async errors handled
- [ ] Collection and field names match exactly what the user provided
