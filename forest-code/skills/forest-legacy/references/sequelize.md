# Sequelize model layer — forest-express (legacy)

Stack: `forest-express-sequelize`. Models live in `/models/<name>.js`.

## Model definition

```javascript
module.exports = (sequelize, DataTypes) => {
  const Company = sequelize.define('companies', {
    name: { type: DataTypes.STRING },
  }, { tableName: 'companies', underscored: true });

  Company.associate = (models) => {
    Company.hasMany(models.orders);
    Company.belongsTo(models.owners, { foreignKey: 'owner_id' });
  };

  return Company;
};
```

Associations: `hasMany`, `belongsTo`, `hasOne`, `belongsToMany(other, { through, foreignKey, otherKey })`.

## Query idioms (inside routes, smart fields, actions)

```javascript
const models = require('../models');

await models.customers.findByPk(id);
await models.customers.findOne({ where: { id } });
await models.customers.findAll({
  where: { id: ids },
  include: [{ model: models.orders, as: 'orders', where: { product_id } }],
  offset, limit,
});
await models.customers.count({ include });

// Raw SQL
await models.sequelize.query(sql, { type: models.sequelize.QueryTypes.SELECT });
```

## Operators in smart-field search / filter

```javascript
const { Op } = models.Sequelize;

// search(query, search): mutate the query's where tree
query.where[Op.and][0][Op.or].push({ firstname: { [Op.like]: `%${search}%` } });
return query;

// filter({ condition, where }): return a where clause
return { firstname: { [Op.like]: `%${condition.value}%` } };
```

## Batching related lookups (avoid N+1 in smart-field `get`)

Use DataLoader keyed by id, resolving with a single `findAll({ where: { id: keys } })`, so a page of records makes one query instead of one per row.
