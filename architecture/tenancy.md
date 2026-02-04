# Tenancy

A tenancy is the container for tenant state. It's not the tenant itself — it's the object that holds the current tenant,
tracks how that tenant was resolved, and provides access to the provider that loaded it.

Understanding the distinction between tenants and tenancies is fundamental to understanding Sprout's design.

## Tenants vs Tenancies

A **tenant** is an entity — a customer, organisation, or isolated context. It's data: a row in a database, an object
with properties. Tenants are passive; they don't do anything.

A **tenancy** is active. It manages the current tenant state for a specific tenant type. It holds:

- The current tenant (or null if none is set)
- The [provider](./tenant-providers.md) used to load tenants of this type
- The [resolver](./identity-resolvers.md) that identified the current tenant (if any)
- The [resolution hook](./resolution-hooks.md) where identification occurred
- Options that control behaviour for this tenancy

The relationship is similar to Laravel's auth guards and users. A user is an entity; a guard manages authentication
state. You can have multiple guards (web, api), each potentially with a different user. Similarly, you can have multiple
tenancies, each potentially with a different tenant.

## Why Tenancies Exist

The simplest multitenancy approach would be a global "current tenant" variable. But this breaks down quickly:

**Multiple tenant types.** A SaaS platform might have organisations and workspaces. An organisation is a tenant. A
workspace within an organisation is also a tenant. These are different types with different providers, different
identifiers, and different scoping rules. A single "current tenant" can't represent both simultaneously.

**Different resolution strategies.** One tenant type might be identified by subdomain, another by path segment, another
by header. Each needs its own resolver configuration.

**Different options per type.** You might want strict tenant validation for one type but lenient handling for another.
Options are per-tenancy, not global.

**Stacking.** When you enter a workspace within an organisation, you need both tenancies active. The organisation
tenancy doesn't disappear — it's still there, still holding its tenant. Sprout maintains a stack of current tenancies.

## Single Tenancy Applications

Most applications have one tenancy. The configuration looks like:

```php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options'  => [
            TenancyOptions::hydrateTenantRelation(),
            TenancyOptions::throwIfNotRelated(),
        ],
    ],
],
```

The tenancy is named `tenants`, uses the provider named `tenants`, and has two options enabled. During a request, this
tenancy gets a tenant loaded into it, and that tenant becomes the "current tenant" for the application.

For single-tenancy applications, you rarely think about the tenancy object directly. You interact with the tenant:

```php
$tenant = tenant();           // Get the current tenant
$tenancy = tenancy();         // Get the current tenancy (rarely needed)
```

## Multiple Tenancies

When you have multiple tenant types, each gets its own tenancy:

```php
'tenancies' => [
    'organizations' => [
        'provider' => 'organizations',
        'options'  => [...],
    ],
    'workspaces' => [
        'provider' => 'workspaces',
        'options'  => [...],
    ],
],
```

Routes can specify which tenancy to resolve:

```php
Route::tenanted('organizations', function () {
    // Organization is the current tenant

    Route::tenanted('workspaces', function () {
        // Both organization AND workspace are current tenants
        // Workspace is the "top" of the stack
    });
});
```

When nested, Sprout maintains a stack. The most recently activated tenancy is "current", but earlier tenancies remain
accessible:

```php
$workspace = tenant();                    // Top of stack (workspace)
$organization = tenant('organizations');  // Specific tenancy by name

$allTenancies = sprout()->getAllCurrentTenancies();  // All active tenancies
```

## How Tenancies Activate

A tenancy becomes active when a tenant is loaded into it. This happens through two methods:

**identify()** takes a public identifier (from a resolver) and uses the provider to load the tenant:

```php
$tenancy->identify('acme');  // Load tenant with identifier 'acme'
```

This dispatches `TenantIdentified` and triggers the full [lifecycle](./tenancy-lifecycle.md) — resolver setup, service overrides, etc.

**load()** takes an internal key and loads directly:

```php
$tenancy->load(42);  // Load tenant with key 42
```

This dispatches `TenantLoaded`. Used primarily for restoring tenant context in queued jobs where you have the key but
not the identifier.

Both methods call the provider, set the tenant, and trigger events. The difference is which retrieval method they use
and which event they dispatch.

## Tenancy State

A tenancy tracks several pieces of state:

**Current tenant** — The loaded tenant entity, or null. Access via `tenant()`, check presence via `check()`.

**Provider** — The tenant provider for this tenancy. Fixed at creation, accessed via `provider()`.

**Resolver** — Which identity resolver identified the tenant. Set when resolution occurs, accessed via `resolver()`.
Null if the tenant was loaded directly (not resolved from a request).

**Resolution hook** — When resolution occurred (Routing or Middleware). Set alongside the resolver, accessed via
`hook()`. Null if not resolved.

**Options** — Configuration flags that affect behaviour. Set at creation from config, can be modified at runtime.

The `wasResolved()` method checks if the current tenant came from request resolution (has both a tenant and a resolver)
versus being set directly.

## Tenancy Options

Options are flags (with optional configuration) that modify tenancy behaviour. They're defined in configuration:

```php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options'  => [
            TenancyOptions::hydrateTenantRelation(),
            TenancyOptions::throwIfNotRelated(),
            TenancyOptions::allOverrides(),
        ],
    ],
],
```

Available options:

**hydrateTenantRelation** — When a model belonging to the tenant is loaded, automatically set its tenant relation to
the current tenant (avoiding an extra query).

**throwIfNotRelated** — When saving a model that should belong to the current tenant but doesn't, throw an exception
rather than silently fixing it.

**allOverrides** — Enable all registered [service overrides](./service-overrides.md) for this tenancy.

**overrides(['cache', 'session'])** — Enable only specific overrides. This takes configuration (the list of overrides),
so it returns an array rather than a string.

Options can be checked and modified at runtime:

```php
$tenancy->hasOption('tenant-relation.hydrate');  // Check
$tenancy->addOption('tenant-relation.strict');   // Add
$tenancy->removeOption('overrides.all');         // Remove
```

The `TenancyOptions` class provides static helpers for checking options:

```php
TenancyOptions::shouldHydrateTenantRelation($tenancy);
TenancyOptions::shouldThrowIfNotRelated($tenancy);
TenancyOptions::shouldEnableOverride($tenancy, 'cache');
```

## The Current Tenancy

Sprout tracks which tenancies are currently active. When a tenancy loads a tenant, it's added to the stack:

```php
sprout()->setCurrentTenancy($tenancy);  // Called internally when tenant loads
```

The "current tenancy" is the top of the stack — the most recently activated:

```php
sprout()->getCurrentTenancy();     // Most recent tenancy with a tenant
sprout()->hasCurrentTenancy();     // Check if any tenancy is active
sprout()->getAllCurrentTenancies(); // All active tenancies
```

When a tenant is cleared (`setTenant(null)`), the tenancy remains in the stack but no longer has a tenant. The stack
is cleared explicitly via `resetTenancies()`, typically at the end of a request.

## Tenancy and the Lifecycle

When a tenancy's tenant changes, it dispatches `CurrentTenantChanged`. This triggers the lifecycle:

1. **SetCurrentTenantContext** — Stores tenant in Laravel's Context (for jobs)
2. **PerformIdentityResolverSetup** — Resolver-specific setup (URL defaults, cookies, etc.)
3. **CleanupServiceOverrides** — Clean up previous tenant's overrides
4. **SetupServiceOverrides** — Set up new tenant's overrides
5. **RefreshTenantAwareDependencies** — Refresh container bindings

The tenancy itself doesn't orchestrate this — it just dispatches the event. Listeners handle the actual work.

See [Tenancy Lifecycle](./tenancy-lifecycle.md) for details on what happens during these events.

## Configuration

Tenancies are configured under `multitenancy.tenancies`:

```php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options'  => [
            TenancyOptions::hydrateTenantRelation(),
            TenancyOptions::throwIfNotRelated(),
            TenancyOptions::allOverrides(),
        ],
    ],
],
```

The default tenancy is specified at `multitenancy.defaults.tenancy`.

Each tenancy must specify a provider (by name). Options are optional.

## Extension

Tenancies are created by `TenancyManager`. The default implementation is `DefaultTenancy`, which handles all standard
use cases.

Custom tenancy implementations are possible by implementing the `Tenancy` contract and registering a custom driver:

```php
$sprout->tenancies()->register('custom', function (array $config, string $name) {
    return new CustomTenancy($name, ...);
});
```

In practice, custom tenancies are rare. The options system and event-driven lifecycle handle most customization needs
without requiring a new tenancy implementation.

## Related Documents

- [Tenant Providers](./tenant-providers.md) — How tenancies load tenants
- [Identity Resolvers](./identity-resolvers.md) — How tenants are identified from requests
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events triggered by tenant changes
- [Service Overrides](./service-overrides.md) — How tenancy options control overrides
