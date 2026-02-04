# Service Overrides

Service overrides are classes that hook into the tenancy lifecycle to perform actions when a tenant is activated or
deactivated. While primarily designed for making services tenant-aware, they are not limited to services — any action
that needs to respond to tenant context changes can be implemented as a service override.

The built-in overrides focus on Laravel's core services (cache, session, filesystem, etc.), but the same mechanism
works equally well for third-party packages or custom application logic.

## Key Concepts

| Term             | Description                                                      |
|------------------|------------------------------------------------------------------|
| Service Override | A class that performs actions on tenant activation/deactivation  |
| Bootable         | An override that requires initialization during framework boot   |
| Stacked Override | A composite override containing multiple sub-overrides           |
| Service Name     | An arbitrary identifier for the override (need not be a service) |
| Driver           | The class implementing the override (specified in configuration) |

## The ServiceOverride Contract

**Location:** `Sprout\Core\Contracts\ServiceOverride`

The core contract that all service overrides must implement.

### Constructor

```php
public function __construct(string $service, array $config);
```

| Parameter  | Type                   | Description                         |
|------------|------------------------|-------------------------------------|
| `$service` | `string`               | The service name (e.g., 'cache')    |
| `$config`  | `array<string, mixed>` | Configuration from overrides config |

### Methods

| Method      | Return Type | Description                         |
|-------------|-------------|-------------------------------------|
| `setup()`   | `void`      | Called when a tenant is activated   |
| `cleanup()` | `void`      | Called when a tenant is deactivated |

### Method: setup()

```php
public function setup(Tenancy $tenancy, Tenant $tenant): void
```

Called when a new tenant is marked as the current tenant. Implementations should perform any actions required when
entering tenant context.

**Parameters:**

| Parameter  | Type      | Description                 |
|------------|-----------|-----------------------------|
| `$tenancy` | `Tenancy` | The tenancy being activated |
| `$tenant`  | `Tenant`  | The tenant being activated  |

### Method: cleanup()

```php
public function cleanup(Tenancy $tenancy, Tenant $tenant): void
```

Called when the current tenant is unset, either to be replaced by another tenant or to exit tenant context entirely.
Called before `setup()` when switching tenants, but only if there was a previous tenant.

**Parameters:**

| Parameter  | Type      | Description                   |
|------------|-----------|-------------------------------|
| `$tenancy` | `Tenancy` | The tenancy being deactivated |
| `$tenant`  | `Tenant`  | The tenant being deactivated  |

## The BootableServiceOverride Contract

**Location:** `Sprout\Core\Contracts\BootableServiceOverride`

An extension of `ServiceOverride` for overrides that require initialization during the framework boot phase.

### Method: boot()

```php
public function boot(Application $app, Sprout $sprout): void
```

Called during the framework boot phase (after all service providers have registered). Used to:

- Extend service managers with custom drivers
- Replace service bindings in the container
- Register event listeners

**Parameters:**

| Parameter | Type          | Description             |
|-----------|---------------|-------------------------|
| `$app`    | `Application` | The Laravel application |
| `$sprout` | `Sprout`      | The Sprout instance     |

## Base Implementation

**Location:** `Sprout\Core\Overrides\BaseOverride`

An abstract base class providing default implementations and common functionality.

### Traits

| Trait           | Purpose                                       |
|-----------------|-----------------------------------------------|
| `AwareOfApp`    | Provides `getApp()`, `setApp()` methods       |
| `AwareOfSprout` | Provides `getSprout()`, `setSprout()` methods |

### Properties

| Property   | Type                   | Description             |
|------------|------------------------|-------------------------|
| `$service` | `string` (readonly)    | The service name        |
| `$config`  | `array<string, mixed>` | The configuration array |

### Methods

| Method        | Return Type            | Description                  |
|---------------|------------------------|------------------------------|
| `getConfig()` | `array<string, mixed>` | Returns the configuration    |
| `setup()`     | `void`                 | Empty default implementation |
| `cleanup()`   | `void`                 | Empty default implementation |

## Stacked Overrides

**Location:** `Sprout\Core\Overrides\StackedOverride`

A composite override that contains multiple sub-overrides, executed in sequence. Implements `BootableServiceOverride`.

### Purpose

Some services require multiple modifications that are logically separate but must be applied together. Stacked overrides
allow grouping these while maintaining separation of concerns.

### Configuration

```php
'filesystem' => [
    'driver'    => \Sprout\Core\Overrides\StackedOverride::class,
    'overrides' => [
        \Sprout\Core\Overrides\FilesystemManagerOverride::class,
        \Sprout\Core\Overrides\FilesystemOverride::class,
    ],
],
```

Sub-overrides can be specified as:

- Class name string (no additional config)
- Array with `driver` key and additional config

```php
'overrides' => [
    SomeOverride::class,
    [
        'driver' => AnotherOverride::class,
        'option' => 'value',
    ],
],
```

### Behaviour

| Phase       | Behaviour                                                   |
|-------------|-------------------------------------------------------------|
| `boot()`    | Creates sub-override instances, boots any that are bootable |
| `setup()`   | Calls `setup()` on each sub-override in order               |
| `cleanup()` | Calls `cleanup()` on each sub-override in order             |

### Methods

| Method           | Return Type                            | Description                        |
|------------------|----------------------------------------|------------------------------------|
| `getOverrides()` | `array<class-string, ServiceOverride>` | Returns all sub-override instances |
| `getOverride()`  | `?ServiceOverride`                     | Returns a specific sub-override    |

## Service Override Manager

**Location:** `Sprout\Core\Managers\ServiceOverrideManager`

Manages the complete lifecycle of service overrides: registration, booting, setup, and cleanup.

### State

| Property             | Type                                         | Description                        |
|----------------------|----------------------------------------------|------------------------------------|
| `$overrides`         | `array<string, ServiceOverride>`             | Registered override instances      |
| `$overrideClasses`   | `array<string, class-string>`                | Mapping of service to driver class |
| `$bootableOverrides` | `list<string>`                               | Services with bootable overrides   |
| `$overridesBooted`   | `bool`                                       | Whether boot phase has completed   |
| `$setupOverrides`    | `array<string, array<class-string, string>>` | Overrides set up per tenancy       |

### Query Methods

| Method                   | Return Type        | Description                                     |
|--------------------------|--------------------|-------------------------------------------------|
| `hasOverride()`          | `bool`             | Check if a service has a registered override    |
| `hasOverrideBooted()`    | `bool`             | Check if a service's override has been booted   |
| `hasOverrideBeenSetUp()` | `bool`             | Check if an override is set up for a tenancy    |
| `hasTenancyBeenSetup()`  | `bool`             | Check if any overrides are set up for a tenancy |
| `isOverrideBootable()`   | `bool`             | Check if a service's override is bootable       |
| `haveOverridesBooted()`  | `bool`             | Check if the boot phase has completed           |
| `getOverrideClass()`     | `?string`          | Get the driver class for a service              |
| `getSetupOverrides()`    | `array`            | Get services set up for a tenancy               |
| `get()`                  | `?ServiceOverride` | Get the override instance for a service         |

### Lifecycle Methods

| Method                | Description                                     |
|-----------------------|-------------------------------------------------|
| `registerOverrides()` | Registers all overrides from configuration      |
| `bootOverrides()`     | Boots all registered bootable overrides         |
| `setupOverrides()`    | Sets up enabled overrides for a tenancy/tenant  |
| `cleanupOverrides()`  | Cleans up set-up overrides for a tenancy/tenant |

### Registration Flow

```
1. registerOverrides() called from SproutServiceProvider::boot()
   └─> For each service in config/sprout/overrides.php:
       └─> register($service)
           └─> Get config for service
           └─> Validate driver class implements ServiceOverride
           └─> Create instance via container
           └─> Inject app/sprout if methods exist
           └─> Store in $overrides
           └─> Dispatch ServiceOverrideRegistered
           └─> If BootableServiceOverride, add to $bootableOverrides
           └─> If boot phase passed, boot immediately
```

### Boot Flow

```
1. bootOverrides() called from SproutServiceProvider (via app->booted callback)
   └─> For each bootable service:
       └─> boot($service)
           └─> Get override instance
           └─> Call boot(app, sprout)
           └─> Dispatch ServiceOverrideBooted
   └─> Set $overridesBooted = true
```

### Setup Flow

```
1. setupOverrides() called from SetupServiceOverrides listener
   └─> Get enabled overrides from TenancyOptions
   └─> Initialize tracking for tenancy
   └─> For each registered override:
       └─> If enabled (allOverrides or in list):
           └─> Call override->setup(tenancy, tenant)
           └─> Track as set up for tenancy
```

### Cleanup Flow

```
1. cleanupOverrides() called from CleanupServiceOverrides listener
   └─> Get enabled overrides from TenancyOptions
   └─> Get set-up overrides for tenancy
   └─> For each set-up override:
       └─> If enabled:
           └─> Call override->cleanup(tenancy, tenant)
           └─> Remove from tracking
       └─> Else:
           └─> Throw ServiceOverrideException::setupButNotEnabled()
   └─> Clear tracking for tenancy
```

## Built-in Overrides

### cache

**Driver:** `CacheOverride`
**Bootable:** Yes

Extends Laravel's cache manager with a `sprout` driver that creates tenant-scoped cache stores.

**Boot Behaviour:**

- Registers a `sprout` driver with the cache manager
- Tracks which cache stores have been created

**Cleanup Behaviour:**

- Forgets all tracked cache stores so they are recreated with new tenant context

**Usage:**

```php
// config/cache.php
'stores' => [
    'tenant' => [
        'driver' => 'sprout',
        'store'  => 'redis',  // The underlying driver
    ],
],
```

### session

**Driver:** `SessionOverride`
**Bootable:** Yes

Extends Laravel's session manager with tenant-aware handlers for file, native, and optionally database drivers.

**Configuration:**

| Option     | Type   | Default | Description                                     |
|------------|--------|---------|-------------------------------------------------|
| `database` | `bool` | `false` | Whether to override the database session driver |

**Boot Behaviour:**

- Extends `file` and `native` drivers with `SproutFileSessionHandler`
- If `database` is true, extends `database` driver with `SproutDatabaseSessionHandler`

**Setup Behaviour:**

- Stores original session config (path, domain, secure, same_site)
- Updates session config from Sprout settings
- Sets tenant-specific session cookie name: `{tenancy}_{identifier}_session`
- Refreshes the session store

**Cleanup Behaviour:**

- Restores original session config
- Refreshes the session store

### cookie

**Driver:** `CookieOverride`
**Bootable:** No

Configures Laravel's CookieJar with tenant-aware defaults.

**Setup Behaviour:**

- Sets default path, domain, secure, and same_site from Sprout settings
- Falls back to session config values if settings not present

### filesystem

**Driver:** `StackedOverride`
**Bootable:** Yes (both sub-overrides)

A stacked override containing:

1. `FilesystemManagerOverride` — Replaces the filesystem manager
2. `FilesystemOverride` — Adds the `sprout` driver

#### FilesystemManagerOverride

**Boot Behaviour:**

- Replaces Laravel's filesystem manager with `SproutFilesystemManager`
- Preserves any already-resolved disks

#### FilesystemOverride

**Boot Behaviour:**

- Registers a `sprout` driver with the filesystem manager
- Tracks which disks have been created

**Cleanup Behaviour:**

- Forgets all tracked disks
- Forgets any disk configured with `driver: sprout`

**Usage:**

```php
// config/filesystems.php
'disks' => [
    'tenant' => [
        'driver' => 'sprout',
        'disk'   => 'local',  // The underlying driver
    ],
],
```

### job

**Driver:** `JobOverride`
**Bootable:** Yes

Enables tenant context propagation to queued jobs.

**Boot Behaviour:**

- Registers `SetCurrentTenantForJob` listener for `JobProcessing` event

**Setup/Cleanup:** None — the listener uses Laravel Context which is set by `SetCurrentTenantContext` bootstrapper.

### auth

**Driver:** `StackedOverride`
**Bootable:** Yes (AuthPasswordOverride only)

A stacked override containing:

1. `AuthGuardOverride` — Resets auth guards on tenant change
2. `AuthPasswordOverride` — Replaces the password broker manager

#### AuthGuardOverride

**Setup/Cleanup Behaviour:**

- Forgets all resolved auth guards
- Guards are lazily re-resolved with new tenant context

#### AuthPasswordOverride

**Boot Behaviour:**

- Replaces `auth.password` binding with `SproutAuthPasswordBrokerManager`
- Removes deferred service registration

**Setup/Cleanup Behaviour:**

- Flushes all resolved password brokers
- Brokers are lazily re-resolved with new tenant context

## Configuration

**Location:** `config/sprout/overrides.php`

### Structure

```php
return [
    'service_name' => [
        'driver' => OverrideClass::class,
        // Additional options passed to constructor
    ],
];
```

### Requirements

| Key      | Required | Description                                 |
|----------|----------|---------------------------------------------|
| `driver` | Yes      | Class implementing `ServiceOverride`        |
| Other    | No       | Passed to override constructor as `$config` |

### Default Configuration

```php
return [
    'filesystem' => [
        'driver'    => StackedOverride::class,
        'overrides' => [
            FilesystemManagerOverride::class,
            FilesystemOverride::class,
        ],
    ],
    'job' => [
        'driver' => JobOverride::class,
    ],
    'cache' => [
        'driver' => CacheOverride::class,
    ],
    'auth' => [
        'driver'    => StackedOverride::class,
        'overrides' => [
            AuthGuardOverride::class,
            AuthPasswordOverride::class,
        ],
    ],
    'cookie' => [
        'driver' => CookieOverride::class,
    ],
    'session' => [
        'driver'   => SessionOverride::class,
        'database' => false,
    ],
];
```

## Enabling Overrides

Overrides must be explicitly enabled per tenancy via `TenancyOptions`.

### Specific Overrides

```php
// config/multitenancy.php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options' => [
            TenancyOptions::overrides(['cache', 'filesystem', 'session']),
        ],
    ],
],
```

### All Overrides

```php
'options' => [
    TenancyOptions::allOverrides(),
],
```

### Query Methods

```php
TenancyOptions::enabledOverrides($tenancy);        // Returns list or null
TenancyOptions::shouldEnableAllOverrides($tenancy); // Returns bool
TenancyOptions::shouldEnableOverride($tenancy, 'cache'); // Returns bool
```

## Events

### ServiceOverrideEvent

**Location:** `Sprout\Core\Events\ServiceOverrideEvent`

Abstract base class for service override events.

| Property    | Type              | Description           |
|-------------|-------------------|-----------------------|
| `$service`  | `string`          | The service name      |
| `$override` | `ServiceOverride` | The override instance |

### ServiceOverrideRegistered

**Location:** `Sprout\Core\Events\ServiceOverrideRegistered`

Dispatched when a service override is registered.

### ServiceOverrideBooted

**Location:** `Sprout\Core\Events\ServiceOverrideBooted`

Dispatched after a bootable service override has been booted.

## Exceptions

### ServiceOverrideException

**Location:** `Sprout\Core\Exceptions\ServiceOverrideException`

| Factory Method         | Condition                                          |
|------------------------|----------------------------------------------------|
| `notBootable()`        | Attempting to boot a non-bootable override         |
| `setupButNotEnabled()` | Override was set up but is not enabled for cleanup |

### MisconfigurationException

| Factory Method    | Condition                                         |
|-------------------|---------------------------------------------------|
| `notFound()`      | Service override not found in configuration       |
| `missingConfig()` | Required config key (e.g., `driver`) is missing   |
| `invalidConfig()` | Driver class does not implement `ServiceOverride` |

## Lifecycle Flow

### Application Boot

```
1. SproutServiceProvider::boot()
   └─> registerServiceOverrides()
       └─> ServiceOverrideManager::registerOverrides()
           └─> For each configured service:
               └─> Create override instance
               └─> Dispatch ServiceOverrideRegistered
               └─> Track bootable overrides

2. Application booted callback
   └─> ServiceOverrideManager::bootOverrides()
       └─> For each bootable override:
           └─> Call boot()
           └─> Dispatch ServiceOverrideBooted
       └─> Mark boot phase complete
```

### Tenant Activation

```
1. Tenant identified/loaded
   └─> CurrentTenantChanged event
       └─> CleanupServiceOverrides listener (if previous tenant)
           └─> ServiceOverrideManager::cleanupOverrides()
       └─> SetupServiceOverrides listener
           └─> ServiceOverrideManager::setupOverrides()
```

### Tenant Deactivation

```
1. Tenancy::setTenant(null)
   └─> CurrentTenantChanged event (current = null)
       └─> CleanupServiceOverrides listener
           └─> ServiceOverrideManager::cleanupOverrides()
       └─> SetupServiceOverrides listener (skipped, no current tenant)
```

## Extension

### Creating a Custom Override

```php
namespace App\Overrides;

use Sprout\Core\Overrides\BaseOverride;
use Sprout\Core\Contracts\Tenancy;
use Sprout\Core\Contracts\Tenant;

class CustomServiceOverride extends BaseOverride
{
    public function setup(Tenancy $tenancy, Tenant $tenant): void
    {
        // Perform actions when tenant is activated
    }

    public function cleanup(Tenancy $tenancy, Tenant $tenant): void
    {
        // Perform actions when tenant is deactivated
    }
}
```

### Creating a Bootable Override

```php
namespace App\Overrides;

use Illuminate\Contracts\Foundation\Application;
use Sprout\Core\Contracts\BootableServiceOverride;
use Sprout\Core\Overrides\BaseOverride;
use Sprout\Core\Sprout;

class CustomBootableOverride extends BaseOverride implements BootableServiceOverride
{
    public function boot(Application $app, Sprout $sprout): void
    {
        // Register drivers, replace bindings, etc.
    }
}
```

### Registering a Custom Override

```php
// config/sprout/overrides.php
return [
    // ... existing overrides

    'custom' => [
        'driver' => \App\Overrides\CustomServiceOverride::class,
        'option' => 'value',
    ],
];
```

### Enabling the Custom Override

```php
// config/multitenancy.php
'tenancies' => [
    'tenants' => [
        'options' => [
            TenancyOptions::overrides(['custom', 'cache', 'filesystem']),
        ],
    ],
],
```

## Related Documents

- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events and bootstrappers that trigger overrides
- [Resolution Hooks](./resolution-hooks.md) — When tenant resolution occurs
- [Identity Resolvers](./identity-resolvers.md) — How tenant identity is extracted
