# Tenancy Lifecycle

The tenancy lifecycle encompasses the events, listeners, and state changes that occur when a tenant is activated,
changed, or deactivated within a tenancy.

## Key Concepts

| Term           | Description                                                        |
|----------------|--------------------------------------------------------------------|
| Tenancy        | An object responsible for managing the state of the current tenant |
| Tenant         | The entity representing a single tenant within the system          |
| Identification | Retrieving a tenant using its public identifier                    |
| Loading        | Retrieving a tenant using its internal key                         |
| Bootstrapper   | A listener that runs when the current tenant changes               |
| Current Tenant | The tenant currently active within a tenancy                       |

## The Tenancy Contract

**Location:** `Sprout\Core\Contracts\Tenancy`

The `Tenancy` contract defines the interface for managing tenant state within a named tenancy context.

### Core Methods

| Method         | Return Type         | Description                            |
|----------------|---------------------|----------------------------------------|
| `getName()`    | `string`            | Get the registered name of the tenancy |
| `check()`      | `bool`              | Check if there is a current tenant     |
| `tenant()`     | `?Tenant`           | Get the current tenant                 |
| `key()`        | `int\|string\|null` | Get the current tenant's key           |
| `identifier()` | `?string`           | Get the current tenant's identifier    |
| `identify()`   | `bool`              | Retrieve and set tenant by identifier  |
| `load()`       | `bool`              | Retrieve and set tenant by key         |
| `setTenant()`  | `static`            | Directly set the current tenant        |
| `provider()`   | `TenantProvider`    | Get the tenant provider                |

### Resolution Tracking Methods

| Method          | Return Type         | Description                                |
|-----------------|---------------------|--------------------------------------------|
| `resolvedVia()` | `static`            | Record which resolver was used             |
| `resolvedAt()`  | `static`            | Record which hook resolution occurred at   |
| `resolver()`    | `?IdentityResolver` | Get the resolver that performed resolution |
| `hook()`        | `?ResolutionHook`   | Get the hook where resolution occurred     |
| `wasResolved()` | `bool`              | Check if the tenant was resolved           |

### Option Methods

| Method              | Return Type           | Description                          |
|---------------------|-----------------------|--------------------------------------|
| `options()`         | `list<string>`        | Get all tenancy options              |
| `hasOption()`       | `bool`                | Check if an option is enabled        |
| `hasOptionConfig()` | `bool`                | Check if an option has configuration |
| `addOption()`       | `static`              | Add an option to the tenancy         |
| `removeOption()`    | `static`              | Remove an option from the tenancy    |
| `optionConfig()`    | `scalar\|array\|null` | Get configuration for an option      |

## The Tenant Contract

**Location:** `Sprout\Core\Contracts\Tenant`

The `Tenant` contract defines the interface that tenant entities must implement.

### Dual Identity System

Tenants have two forms of identity:

| Identity   | Method                  | Purpose                                        |
|------------|-------------------------|------------------------------------------------|
| Identifier | `getTenantIdentifier()` | Public-facing identity (e.g., slug, subdomain) |
| Key        | `getTenantKey()`        | Internal identity (e.g., database primary key) |

**Rationale:** The identifier is used in URLs, subdomains, and other public-facing contexts where a human-readable,
stable value is preferred. The key is used for internal operations such as database relationships and job processing
where immutability and performance are priorities.

### Methods

| Method                      | Return Type   | Description                             |
|-----------------------------|---------------|-----------------------------------------|
| `getTenantIdentifier()`     | `string`      | Get the public identifier               |
| `getTenantIdentifierName()` | `string`      | Get the storage name for the identifier |
| `getTenantKey()`            | `int\|string` | Get the internal key                    |
| `getTenantKeyName()`        | `string`      | Get the storage name for the key        |

## The TenantHasResources Contract

**Location:** `Sprout\Core\Contracts\TenantHasResources`

An optional companion contract for `Tenant` that marks a tenant as having dedicated resources (files, storage paths,
etc.) identified by a resource key.

### Purpose

The resource key provides a stable, filesystem-safe identifier for tenant-specific resources. While the tenant
identifier may contain characters unsuitable for file paths or may change over time, the resource key is designed to
remain constant and safe for use in:

- Filesystem paths (e.g., `storage/tenants/{resource_key}/`)
- Cache prefixes
- Queue names
- Any context requiring a stable, unique string

### Methods

| Method                      | Return Type | Description                             |
|-----------------------------|-------------|-----------------------------------------|
| `getTenantResourceKey()`    | `string`    | Get the resource key                    |
| `getTenantResourceKeyName()`| `string`    | Get the storage name for the resource key |

### Provider Support

The `TenantProvider` contract includes a method for retrieving tenants by their resource key:

```php
public function retrieveByResourceKey(string $resourceKey): (Tenant&TenantHasResources)|null
```

This method requires the tenant class to implement `TenantHasResources`.

## Tenant Retrieval Methods

The tenancy provides two methods for retrieving and setting the current tenant:

### identify()

Retrieves a tenant using its public identifier.

```php
public function identify(string $identifier): bool
```

**Flow:**

1. Calls `$provider->retrieveByIdentifier($identifier)`
2. If no tenant found, clears resolver and hook, returns `false`
3. Calls `setTenant($tenant)`
4. Dispatches `TenantIdentified` event
5. Returns `true`

**Use Case:** Request-based tenant resolution where the identifier comes from a URL, subdomain, header, etc.

### load()

Retrieves a tenant using its internal key.

```php
public function load(int|string $key): bool
```

**Flow:**

1. Calls `$provider->retrieveByKey($key)`
2. If no tenant found, clears resolver and hook, returns `false`
3. Calls `setTenant($tenant)`
4. Dispatches `TenantLoaded` event
5. Returns `true`

**Use Case:** Job processing, scheduled tasks, or any context where the tenant key is stored (e.g., Laravel Context).

### setTenant()

Directly sets the current tenant without retrieval.

```php
public function setTenant(?Tenant $tenant): static
```

**Flow:**

1. Stores the previous tenant
2. If the tenant has changed, updates `$this->tenant` and dispatches `CurrentTenantChanged`
3. If the new tenant is `null`, clears resolver and hook
4. Returns `$this`

## Events

### Event Hierarchy

```
TenantFound (abstract)
├── TenantIdentified — Dispatched by identify()
└── TenantLoaded     — Dispatched by load()

CurrentTenantChanged — Dispatched by setTenant() when tenant changes
```

### TenantFound

**Location:** `Sprout\Core\Events\TenantFound`

Abstract base class for tenant retrieval events.

| Property   | Type      | Description               |
|------------|-----------|---------------------------|
| `$tenant`  | `Tenant`  | The tenant that was found |
| `$tenancy` | `Tenancy` | The tenancy that found it |

### TenantIdentified

**Location:** `Sprout\Core\Events\TenantIdentified`

Dispatched when a tenant is retrieved via `identify()` (using its public identifier).

### TenantLoaded

**Location:** `Sprout\Core\Events\TenantLoaded`

Dispatched when a tenant is retrieved via `load()` (using its internal key).

### CurrentTenantChanged

**Location:** `Sprout\Core\Events\CurrentTenantChanged`

Dispatched when the current tenant for a tenancy changes to a different value, including transitions to or from `null`.

| Property    | Type      | Description                      |
|-------------|-----------|----------------------------------|
| `$tenancy`  | `Tenancy` | The tenancy whose tenant changed |
| `$previous` | `?Tenant` | The previous tenant (or null)    |
| `$current`  | `?Tenant` | The current tenant (or null)     |

**Dispatch Condition:** Only dispatched when `$previousTenant !== $tenant`.

## Bootstrappers

Bootstrappers are event listeners registered for `CurrentTenantChanged` that perform setup and teardown actions when the
tenant changes.

### Configuration

**Location:** `config/sprout/core.php`

```php
'bootstrappers' => [
    \Sprout\Core\Listeners\SetCurrentTenantContext::class,
    \Sprout\Core\Listeners\PerformIdentityResolverSetup::class,
    \Sprout\Core\Listeners\CleanupServiceOverrides::class,
    \Sprout\Core\Listeners\SetupServiceOverrides::class,
    \Sprout\Core\Listeners\RefreshTenantAwareDependencies::class,
],
```

### Registration

Bootstrappers are registered in `SproutServiceProvider::registerTenancyBootstrappers()`:

```php
foreach ($bootstrappers as $bootstrapper) {
    $events->listen(CurrentTenantChanged::class, $bootstrapper);
}
```

### Execution Order

Bootstrappers execute in the order they appear in configuration:

| Order | Listener                         | Purpose                                    |
|-------|----------------------------------|--------------------------------------------|
| 1     | `SetCurrentTenantContext`        | Store tenant key in Laravel Context        |
| 2     | `PerformIdentityResolverSetup`   | Execute resolver-specific setup            |
| 3     | `CleanupServiceOverrides`        | Revert previous tenant's service overrides |
| 4     | `SetupServiceOverrides`          | Apply current tenant's service overrides   |
| 5     | `RefreshTenantAwareDependencies` | Update container bindings                  |

## Built-in Bootstrappers

### SetCurrentTenantContext

**Location:** `Sprout\Core\Listeners\SetCurrentTenantContext`

Stores the current tenant's key in Laravel's Context service for propagation to queued jobs.

**Behaviour:**

- Maintains a `sprout.tenants` array in Context keyed by tenancy name
- If `$event->current` is `null`, removes the entry for this tenancy
- If `$event->current` is set, stores `$tenancy->getName() => $tenant->getTenantKey()`

**Purpose:** Enables tenant context propagation to queued jobs via Laravel's Context service.

### PerformIdentityResolverSetup

**Location:** `Sprout\Core\Listeners\PerformIdentityResolverSetup`

Calls the `setup()` method on the identity resolver that performed resolution.

**Behaviour:**

```php
$event->tenancy->resolver()?->setup($event->tenancy, $event->current);
```

**Purpose:** Allows resolvers to perform post-identification actions such as:

- Setting URL defaults for parameter-based resolvers
- Storing tenant identifier in cookies
- Storing tenant identifier in session

### CleanupServiceOverrides

**Location:** `Sprout\Core\Listeners\CleanupServiceOverrides`

Reverts service overrides that were applied for the previous tenant.

**Behaviour:**

- Only executes if `$event->previous !== null`
- Delegates to `ServiceOverrideManager::cleanupOverrides()`

**Purpose:** Ensures services are reset to their default state before applying new tenant's overrides.

### SetupServiceOverrides

**Location:** `Sprout\Core\Listeners\SetupServiceOverrides`

Applies service overrides for the current tenant.

**Behaviour:**

- Only executes if `$event->current !== null`
- Delegates to `ServiceOverrideManager::setupOverrides()`

**Purpose:** Makes Laravel services tenant-aware (cache, filesystem, session, etc.).

### RefreshTenantAwareDependencies

**Location:** `Sprout\Core\Listeners\RefreshTenantAwareDependencies`

Triggers container refresh for tenant-aware dependencies.

**Behaviour:**

- Only executes if `$event->current !== null`
- Calls `forgetExtenders()` and `extend()` on the `Tenant` binding

**Purpose:** Ensures classes implementing `TenantAware` receive the updated tenant instance.

## Job Tenant Context

### SetCurrentTenantForJob

**Location:** `Sprout\Core\Listeners\SetCurrentTenantForJob`

An event listener for `JobProcessing` that restores tenant context when processing queued jobs.

**Registration:** This listener is registered by the job service override, not in the core bootstrappers.

**Behaviour:**

1. Retrieves `sprout.tenants` from Laravel Context
2. For each tenancy/key pair, calls `$tenancy->load($key)`
3. Sets the tenancy as current via `$sprout->setCurrentTenancy()`

**Note:** Uses `load()` rather than `identify()` because Context stores the tenant key, not the identifier.

## Tenant-Aware Dependencies

### TenantAware Contract

**Location:** `Sprout\Core\Contracts\TenantAware`

An interface for classes that need to be notified when the tenant or tenancy changes.

| Method                | Return Type | Description                                 |
|-----------------------|-------------|---------------------------------------------|
| `shouldBeRefreshed()` | `bool`      | Whether to receive updates on tenant change |
| `getTenant()`         | `?Tenant`   | Get the current tenant                      |
| `hasTenant()`         | `bool`      | Check if there is a tenant                  |
| `setTenant()`         | `static`    | Set the tenant (called on change)           |
| `getTenancy()`        | `?Tenancy`  | Get the current tenancy                     |
| `hasTenancy()`        | `bool`      | Check if there is a tenancy                 |
| `setTenancy()`        | `static`    | Set the tenancy (called on change)          |

### Registration

In `SproutServiceProvider::registerTenantAwareHandling()`:

```php
$this->app->afterResolving(TenantAware::class, function (TenantAware $tenantAware) {
    if ($tenantAware->shouldBeRefreshed()) {
        $this->app->refresh(Tenant::class, $tenantAware, 'setTenant');
        $this->app->refresh(Tenancy::class, $tenantAware, 'setTenancy');
    }
});
```

## Tenancy Options

**Location:** `Sprout\Core\TenancyOptions`

A helper class providing named options and query methods for tenancy configuration.

### Available Options

| Option                    | Method                    | Purpose                                        |
|---------------------------|---------------------------|------------------------------------------------|
| `tenant-relation.hydrate` | `hydrateTenantRelation()` | Auto-hydrate tenant relations on models        |
| `tenant-relation.strict`  | `throwIfNotRelated()`     | Throw exception if model not related to tenant |
| `overrides`               | `overrides(array)`        | Specify which overrides to enable              |
| `overrides.all`           | `allOverrides()`          | Enable all registered overrides                |

### Query Methods

| Method                          | Return Type | Description                                    |
|---------------------------------|-------------|------------------------------------------------|
| `shouldHydrateTenantRelation()` | `bool`      | Check if tenant relations should auto-hydrate  |
| `shouldThrowIfNotRelated()`     | `bool`      | Check if mismatched tenant should throw        |
| `enabledOverrides()`            | `?array`    | Get list of enabled override names             |
| `shouldEnableAllOverrides()`    | `bool`      | Check if all overrides should be enabled       |
| `shouldEnableOverride()`        | `bool`      | Check if a specific override should be enabled |

### Configuration Example

```php
// config/multitenancy.php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options' => [
            TenancyOptions::hydrateTenantRelation(),
            TenancyOptions::throwIfNotRelated(),
            TenancyOptions::overrides(['cache', 'filesystem']),
        ],
    ],
],
```

## Default Implementation

**Location:** `Sprout\Core\Support\DefaultTenancy`

The default implementation of the `Tenancy` contract.

### Constructor

```php
public function __construct(
    string $name,
    TenantProvider $provider,
    array $options
)
```

### State

| Property        | Type                | Description                          |
|-----------------|---------------------|--------------------------------------|
| `$name`         | `string`            | The tenancy name                     |
| `$provider`     | `TenantProvider`    | The tenant provider                  |
| `$resolver`     | `?IdentityResolver` | The resolver used for identification |
| `$tenant`       | `?Tenant`           | The current tenant                   |
| `$options`      | `array`             | List of enabled options              |
| `$optionConfig` | `array`             | Configuration for options            |
| `$hook`         | `?ResolutionHook`   | The hook where resolution occurred   |

### Factory

Created by `TenancyManager::createDefaultTenancy()`:

```php
protected function createDefaultTenancy(array $config, string $name): DefaultTenancy
{
    return new DefaultTenancy(
        $name,
        $this->providerManager->get($config['provider'] ?? null),
        $config['options'] ?? []
    );
}
```

## Lifecycle Flow

### Request-Based Resolution

```
1. Request arrives
2. Resolution hook triggers (Routing or Middleware)
3. ResolutionHelper::handleResolution() called
   └─> $tenancy->resolvedVia($resolver)->resolvedAt($hook)
   └─> Identity extracted from request
   └─> $tenancy->identify($identity)
       └─> Provider retrieves tenant
       └─> $tenancy->setTenant($tenant)
           └─> CurrentTenantChanged dispatched
               └─> Bootstrappers execute in order
       └─> TenantIdentified dispatched
```

### Job-Based Resolution

```
1. Job begins processing
2. JobProcessing event fires
3. SetCurrentTenantForJob listener handles event
   └─> Retrieves tenant keys from Context
   └─> For each tenancy:
       └─> $tenancy->load($key)
           └─> Provider retrieves tenant
           └─> $tenancy->setTenant($tenant)
               └─> CurrentTenantChanged dispatched
                   └─> Bootstrappers execute in order
           └─> TenantLoaded dispatched
       └─> $sprout->setCurrentTenancy($tenancy)
```

### Tenant Change

```
1. $tenancy->setTenant($newTenant) called
2. If $newTenant !== $previousTenant:
   └─> CurrentTenantChanged dispatched
       └─> SetCurrentTenantContext updates Context
       └─> PerformIdentityResolverSetup calls resolver setup
       └─> CleanupServiceOverrides reverts previous overrides
       └─> SetupServiceOverrides applies new overrides
       └─> RefreshTenantAwareDependencies updates bindings
3. If $newTenant === null:
   └─> Resolver and hook cleared
```

### Tenant Removal

```
1. $tenancy->setTenant(null) called
2. CurrentTenantChanged dispatched (current = null)
   └─> SetCurrentTenantContext removes from Context
   └─> PerformIdentityResolverSetup called with null
   └─> CleanupServiceOverrides reverts overrides
   └─> SetupServiceOverrides skipped (no current tenant)
   └─> RefreshTenantAwareDependencies skipped (no current tenant)
3. Resolver and hook cleared
```

## Related Documents

- [Resolution Hooks](./resolution-hooks.md) — When and how tenant resolution occurs
- [Identity Resolvers](./identity-resolvers.md) — Extracting tenant identity from requests
- [Service Overrides](./service-overrides.md) — Making Laravel services tenant-aware
