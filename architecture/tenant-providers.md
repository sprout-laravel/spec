# Tenant Providers

Tenant providers retrieve tenant entities from storage. They form the bridge
between [identity resolvers](./identity-resolvers.md) — which extract
raw identifier strings from requests — and the tenant entities that represent those tenants in code.

The key design principle is **separation of concerns**: resolvers know how to find identifiers in requests but nothing
about storage. Providers know how to load tenants from storage but nothing about where the identifier came from. This
means any resolver can work with any provider, and both can be swapped independently.

## The Dual Identity System

Tenants have two distinct identities, and understanding why is central to how providers work.

### Identifier vs Key

The **identifier** is a public-facing value — what appears in subdomains, URL paths, headers, or cookies. It's
human-readable and often chosen by the tenant themselves: `acme`, `acme-corp`, `my-company`.

The **key** is an internal value — typically the database primary key. It's stable, efficient for lookups and
relationships, and never exposed publicly: `1`, `42`, `550e8400-e29b-41d4-a716-446655440000`.

### Why Both?

The separation exists because public identifiers need to change while internal references shouldn't.

Consider a tenant that rebrands from "Acme Corp" to "Initech". They want their subdomain to change from `acme` to
`initech`. With a single identity, every database relationship, every queued job, every cached reference would break.
With dual identity, only the identifier changes — the key remains stable and all internal references continue working.

This also enables different identifier schemes per context. The same tenant might be `acme` in subdomains,
`acme-corporation` in URL paths (where hyphens are preferred), and `ACME001` in API headers (where codes are expected).
The key stays consistent regardless.

### Resource Keys

Some tenants have dedicated resources — files, directories, storage paths. The **resource key** is a third identity
specifically for this purpose.

Why not use the identifier? Because identifiers might change (the rebrand scenario), but you don't want to move
terabytes of files when a tenant changes their subdomain. Resource keys are typically UUIDs or other stable identifiers
that remain constant throughout the tenant's lifetime.

Resource keys are optional. Not all tenants have dedicated resources, so the `TenantHasResources` contract is separate
from the core `Tenant` contract. Providers handle both cases — tenants with resources can be looked up by resource key,
tenants without simply don't support that lookup method.

## Provider Types

Sprout includes two provider implementations, each with different trade-offs.

### Eloquent Provider

Uses Eloquent models for tenant retrieval. Choose this when:

- Your tenant is already an Eloquent model
- You want model features: events, relationships, accessors, mutators, casting
- You need to eager-load relationships during tenant resolution
- Consistency with the rest of your Laravel application matters

The trade-off is weight. Eloquent adds overhead — model instantiation, attribute casting, event dispatching. For most
applications this is negligible, but it exists.

### Database Provider

Uses Laravel's query builder directly, bypassing Eloquent entirely. Choose this when:

- You want the lightest possible tenant lookup
- Tenants come from a legacy table that doesn't map well to Eloquent
- You're loading tenants from a different database or connection
- You only need basic attribute access, not model features

The provider returns entity objects (by default `GenericTenant`, or a custom class you specify). These are simple
objects with array-backed attribute storage — no events, no relationships, no casting.

The database provider also accepts an Eloquent model class for the `table` config. It won't use Eloquent, but it will
read the table name and connection from the model. This is useful when you want a lightweight fallback provider that
matches your Eloquent model's storage location.

## How Providers Fit Into Resolution

```
Request
   │
   ▼
Identity Resolver
   extracts identifier string ("acme")
   │
   ▼
Tenant Provider
   loads tenant entity by identifier
   │
   ▼
Tenancy
   holds the loaded tenant, triggers lifecycle events
```

The provider doesn't know or care how the identifier was obtained. Subdomain, path segment, HTTP header, cookie,
session — it's all the same to the provider. It receives a string and returns a tenant (or null).

This decoupling means:

- The same provider works with any resolver
- You can switch resolvers without touching provider configuration
- Custom resolvers automatically work with built-in providers
- Custom providers automatically work with built-in resolvers

## Retrieval Methods

Providers support three ways to retrieve tenants, each serving a different purpose.

**By identifier** is the primary path. Identity resolvers extract identifiers from requests, and providers load tenants
by those identifiers. This is how tenant resolution works during normal HTTP requests.

**By key** is for internal operations. When a queued job needs to restore tenant context, it stores and retrieves the
tenant key (not the identifier). Keys are stable and efficient — exactly what you need for background processing.

**By resource key** is for resource operations. When the filesystem override needs to determine which tenant owns a
particular directory, it looks up tenants by resource key. This only works for tenants implementing
`TenantHasResources`.

## Configuration

Providers are configured under `multitenancy.providers`:

```php
'providers' => [
    'tenants' => [
        'driver' => 'eloquent',
        'model'  => App\Models\Tenant::class,
    ],
],
```

For the database provider:

```php
'providers' => [
    'tenants' => [
        'driver'     => 'database',
        'table'      => 'tenants',
        'connection' => 'tenant_db',  // optional
        'entity'     => App\Entities\Tenant::class,  // optional
    ],
],
```

Tenancies reference providers by name:

```php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',  // references the provider above
    ],
],
```

Multiple tenancies can share a provider, or each can have its own.

## Extension

Custom providers extend the system for cases the built-in providers don't cover.

### When to Create a Custom Provider

- Tenants come from an external API
- You need caching between the provider and storage
- Tenant data is distributed across multiple sources
- You want to add validation or transformation during loading

### The Pattern

Extend `BaseTenantProvider` and implement the three retrieval methods. The base class handles name storage and provides
access to the Laravel application container.

Register custom drivers with the provider manager:

```php
$sprout->providers()->register('api', function (array $config, string $name) {
    return new ApiTenantProvider($name, $config['base_url']);
});
```

Then use it in configuration:

```php
'providers' => [
    'external' => [
        'driver'   => 'api',
        'base_url' => 'https://api.example.com',
    ],
],
```

## Constraints and Gotchas

**Resource key lookups require TenantHasResources.** If you call `retrieveByResourceKey()` on a provider whose tenant
doesn't implement this contract, you'll get a `MisconfigurationException`. The provider validates this at runtime, not
configuration time.

**Providers don't validate tenant state.** A provider returns whatever it finds in storage. If a tenant is soft-deleted,
disabled, or otherwise invalid, the provider still returns it. Validation happens elsewhere — typically in middleware or
application logic.

**Identifier uniqueness is a storage concern.** Providers assume identifiers are unique but don't enforce it. If your
database allows duplicate identifiers, the provider returns the first match. Enforce uniqueness at the database level.

**Eloquent provider caches the model instance.** For efficiency, `EloquentTenantProvider` creates one model instance and
reuses it for building queries. This is usually fine but means the model shouldn't carry request-specific state.

## Related Documents

- [Tenancy](./tenancy.md) — How tenancies hold loaded tenants
- [Identity Resolvers](./identity-resolvers.md) — How identifiers are extracted from requests
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — What happens after a tenant is loaded
- [Service Overrides](./service-overrides.md) — How services become tenant-aware
