# Tenant-Aware Models & Eloquent Integration

Sprout integrates deeply with Laravel's Eloquent ORM to make models tenant-aware. This integration ensures that queries
automatically scope to the current tenant, models are associated with tenants on creation, and tenant relationships are
handled consistently throughout the application.

## The Problem

In a multitenant application, most database queries need to be scoped to the current tenant. Without automatic scoping,
every query must manually include a `where` clause:

```php
// Without automatic scoping — error-prone and repetitive
$posts = Post::where('tenant_id', $currentTenant->id)->get();
```

This approach has several problems:

- **Easy to forget.** Missing a single `where` clause leaks data between tenants.
- **Scattered logic.** Tenant filtering is duplicated throughout the codebase.
- **Relationship leakage.** Eager loading and relationship queries might bypass manual filters.
- **Manual assignment.** Creating models requires remembering to set the tenant foreign key.

Sprout solves this with a layered system: traits declare intent, scopes filter queries, and observers handle model
creation.

## When You Need This

This Eloquent integration is designed for **shared database architectures** — where all tenants' data lives in the same
database, distinguished by foreign keys.

If you're using [Seedling](../rfcs/0001-seedling.md) to give each tenant their own database connection, you don't need
these traits and scopes. When queries run against a tenant-specific database, there's no risk of cross-tenant data
leakage — the database itself only contains that tenant's data. Tenant isolation is handled at the connection level, not
the query level.

| Architecture             | Isolation mechanism            | Need these traits?     |
|--------------------------|--------------------------------|------------------------|
| Shared database          | Query scoping via foreign keys | Yes                    |
| Tenant-specific database | Separate database per tenant   | No                     |
| Hybrid                   | Mix of both                    | Only for shared tables |

Most applications using tenant-specific databases can use plain Eloquent models without any Sprout traits. Your tenant
model still needs to implement the `Tenant` contract (and `IsTenant` provides a convenient baseline implementation), but
`BelongsToTenant` and related traits are unnecessary for tenant-owned data.

## Architecture Overview

The Eloquent integration has three layers that work together:

```
┌─────────────────────────────────────────────────────────────┐
│                         Traits                              │
│   (BelongsToTenant, BelongsToManyTenants, IsTenantChild)   │
│                  Declare relationships                      │
└─────────────────────────┬───────────────────────────────────┘
                          │ boots
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Global Scopes                           │
│ (BelongsToTenantScope, BelongsToManyTenantsScope, etc.)    │
│              Filter all queries automatically               │
└─────────────────────────┬───────────────────────────────────┘
                          │ registered alongside
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       Observers                             │
│   (BelongsToTenantObserver, BelongsToManyTenantsObserver)  │
│           Associate models with tenant on create            │
└─────────────────────────────────────────────────────────────┘
```

**Traits** are what you use in your models. They declare how a model relates to tenants and boot the appropriate scopes
and observers.

**Scopes** intercept every query on the model, adding tenant filtering automatically. They work on `SELECT`, `UPDATE`,
and `DELETE` queries.

**Observers** handle model lifecycle events. When a model is created, they automatically associate it with the current
tenant.

## Two Relationship Patterns

Models can relate to tenants in two ways, and the choice affects how scoping and association work.

### BelongsToTenant — Exclusive Ownership

A model belongs to exactly one tenant. This is the most common pattern for tenant-specific data.

```php
class Post extends Model
{
    use BelongsToTenant;
}
```

The model has a foreign key column (typically `tenant_id`) pointing to a single tenant. Queries are scoped by that
column. When created, the foreign key is set to the current tenant's key.

Use this for: posts, orders, invoices, user profiles — anything owned by one tenant.

### BelongsToManyTenants — Shared Ownership

A model can belong to multiple tenants through a pivot table. This is less common but necessary for shared resources.

```php
class Template extends Model
{
    use BelongsToManyTenants;
}
```

The model uses a many-to-many relationship. Queries are scoped through a `whereHas` on the relationship. When created,
a pivot record is inserted for the current tenant.

Use this for: shared templates, global resources with tenant access, content that can be assigned to multiple tenants.

### Choosing Between Them

The choice is about data ownership:

| Aspect     | BelongsToTenant        | BelongsToManyTenants        |
|------------|------------------------|-----------------------------|
| Ownership  | Exclusive — one tenant | Shared — multiple tenants   |
| Storage    | Foreign key column     | Pivot table                 |
| Query cost | Simple `WHERE` clause  | `EXISTS` subquery           |
| Creation   | Sets foreign key       | Inserts pivot record        |
| Use case   | Most tenant data       | Shared/assignable resources |

Most applications use `BelongsToTenant` for nearly everything. `BelongsToManyTenants` is for specific scenarios where
sharing genuinely makes sense.

## How Automatic Scoping Works

When you use a tenant-aware trait, it boots a global scope that modifies every query on that model.

### Query Modification

The scope adds tenant filtering to the query:

```php
// What you write
$posts = Post::where('published', true)->get();

// What actually executes (with BelongsToTenant)
$posts = Post::where('published', true)
    ->where('tenant_id', $currentTenant->getKey())
    ->get();

// What actually executes (with BelongsToManyTenants)
$posts = Template::where('active', true)
    ->whereHas('tenants', fn ($q) => $q->where('id', $currentTenant->getKey()))
    ->get();
```

This happens automatically. You don't need to remember to add the filter — it's always there.

### Relationship Discovery

The scopes need to know which column or relationship to filter by. Rather than requiring explicit configuration, they
use reflection to discover this automatically.

For `BelongsToTenant`, the scope looks for a method with the `#[TenantRelation]` attribute:

```php
class Post extends Model
{
    use BelongsToTenant;

    #[TenantRelation]
    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class);
    }
}
```

If no attributed method exists, it falls back to calling a method named after the tenancy (e.g., `tenant()` for the
default tenancy).

This design means:

- Simple cases work without any configuration
- Complex cases (custom relationship names, multiple tenancies) are supported via the attribute
- The foreign key is derived from the relationship, not hardcoded

### When No Tenant Is Active

If queries run without an active tenant, the scope's behaviour depends on the model:

**Standard models** throw an exception. Running `Post::all()` without a tenant is almost certainly a bug — you'd be
querying all tenants' data. The exception prevents accidental data leakage.

**Optional tenant models** (implementing `OptionalTenant`) allow queries without a tenant. The scope silently does
nothing, returning all records. Use this for models that genuinely need to work both inside and outside tenant context.

```php
class AuditLog extends Model implements OptionalTenant
{
    use BelongsToTenant;
}

// Works without a tenant — returns all audit logs
$logs = AuditLog::all();
```

## Automatic Association on Create

When you create a tenant-aware model, observers automatically associate it with the current tenant.

### BelongsToTenant Models

The observer sets the foreign key:

```php
// What you write
$post = Post::create(['title' => 'Hello']);

// What happens internally
$post = new Post(['title' => 'Hello']);
$post->tenant()->associate($currentTenant);  // Observer does this
$post->save();
```

You don't need to manually set `tenant_id` — it happens automatically.

### BelongsToManyTenants Models

The observer attaches the tenant via the pivot:

```php
// What you write
$template = Template::create(['name' => 'Invoice']);

// What happens internally
$template = Template::create(['name' => 'Invoice']);
$template->tenants()->attach($currentTenant);  // Observer does this
```

The pivot record is created in the `created` event, after the model exists.

### Strict Mode

By default, creating a tenant-aware model without an active tenant throws an exception. This prevents orphaned records
that belong to no tenant.

You can disable this with a tenancy option:

```php
'tenancies' => [
    'tenants' => [
        'options' => [
            TenancyOptions::throwIfNotRelated(false),
        ],
    ],
],
```

With strict mode disabled, models created without a tenant simply won't have the association set. This is rarely what
you want, but it's available for edge cases.

## Relationship Hydration

When a tenant becomes active, Sprout can optionally hydrate the tenant relationship on all tenant-aware models. This
means `$post->tenant` returns the current tenant without hitting the database.

Enable via tenancy options:

```php
'options' => [
    TenancyOptions::hydrateTenantRelation(),
],
```

With hydration enabled:

```php
$post = Post::first();
$post->tenant;  // Returns current tenant, no query executed
```

This works because within tenant context, all `BelongsToTenant` models belong to the current tenant by definition. The
scope guarantees it. Hydration exploits this guarantee to avoid redundant queries.

Hydration happens at the [tenancy lifecycle](./tenancy-lifecycle.md) level — when `CurrentTenantChanged` fires, the
system updates relationships on relevant models.

## The Tenant Model Itself

The entity that represents a tenant also needs traits:

```php
class Tenant extends Model implements \Sprout\Contracts\Tenant
{
    use IsTenant;
}
```

`IsTenant` provides the contract implementation and relationship helpers. It's separate from `BelongsToTenant` because
the tenant model doesn't belong to a tenant — it *is* the tenant.

### Tenant Children

For hierarchical tenant data (a tenant's settings, preferences, or profile), use `IsTenantChild`:

```php
class TenantSettings extends Model
{
    use IsTenantChild;

    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class);
    }
}
```

`IsTenantChild` is similar to `BelongsToTenant` but designed for models that are part of the tenant's own structure
rather than tenant-owned business data. The distinction is conceptual — both scope queries to the current tenant.

### Tenant Resources

When a tenant has associated resources (like a database connection or storage path), implement `TenantHasResources`:

```php
class Tenant extends Model implements \Sprout\Contracts\Tenant, TenantHasResources
{
    use IsTenant, HasTenantResources;

    public function getTenantResourceKey(): string
    {
        return $this->slug;  // Used in paths like storage/tenants/{slug}/
    }
}
```

This is used by [service overrides](./service-overrides.md) when they need a tenant-specific identifier for resources
like filesystem paths or cache prefixes.

## Configuration

### Tenancy Options

Model behaviour is controlled through [tenancy options](./tenancy.md):

```php
'tenancies' => [
    'tenants' => [
        'options' => [
            // Hydrate tenant relationships automatically
            TenancyOptions::hydrateTenantRelation(),

            // Throw exception when creating models without a tenant (default: true)
            TenancyOptions::throwIfNotRelated(true),
        ],
    ],
],
```

### Relationship Configuration

The `#[TenantRelation]` attribute accepts a tenancy name for multi-tenancy scenarios:

```php
class Post extends Model
{
    use BelongsToTenant;

    // This relationship is for the 'organizations' tenancy
    #[TenantRelation('organizations')]
    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }

    // This relationship is for the default tenancy
    #[TenantRelation]
    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class);
    }
}
```

This allows a single model to be scoped by different tenants depending on which tenancy is active.

## Constraints and Gotchas

**Scopes apply to all query types.** Updates and deletes are also scoped. `Post::where('id', 5)->delete()` only deletes
the post if it belongs to the current tenant. This is usually correct, but be aware of it.

**Eager loading respects scopes.** When you eager load a tenant-aware relationship, the scope applies. This prevents
accidental cross-tenant data in nested relationships.

**`withoutGlobalScope()` bypasses protection.** Laravel allows removing global scopes. If you do this on a tenant-aware
model, you're bypassing tenant isolation. Use with extreme caution and only when you genuinely need cross-tenant access.

**Relationship hydration is optional.** It's not enabled by default because it adds overhead during tenant changes.
Enable it when you frequently access `$model->tenant` and want to avoid the queries.

**Optional tenants need explicit marking.** A model doesn't become optional just by allowing nullable foreign keys. You
must implement `OptionalTenant` for the scope to permit tenant-less queries.

**Observers run in `creating`/`created` events.** If you have your own observers that need the tenant association, they
should run after Sprout's observers, or access the tenant directly via the tenancy rather than the relationship.

**Multi-tenancy requires attributes.** If you have multiple tenancies and a model that participates in more than one,
you must use `#[TenantRelation('tenancy_name')]` to specify which relationship belongs to which tenancy.

## Related Documents

- [Tenancy](./tenancy.md) — Options that control model behaviour
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — When relationship hydration occurs
- [Tenant Providers](./tenant-providers.md) — How tenants are loaded before scoping applies
- [Service Overrides](./service-overrides.md) — How TenantHasResources is used
