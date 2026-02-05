# Managers & Factories

Sprout uses a consistent factory pattern across its core components. Managers create, cache, and provide access to
configured instances of tenancies, providers, resolvers, and overrides. This pattern enables extensibility without
modifying core code.

## The Pattern

Each manager is both a **factory** (creates instances from configuration) and a **registry** (caches and retrieves
them). The pattern solves several problems:

- **Multiple implementations** — Subdomain, path, header resolvers; Eloquent, database providers
- **Configuration-driven** — Instances are created based on config, not hardcoded
- **Extensibility** — Third-party packages can register custom drivers
- **Lazy creation** — Instances are only created when first requested
- **Instance reuse** — Once created, instances are cached and reused

## Managers Overview

Sprout has four managers:

| Manager                                            | Creates                    | Config Key               |
|----------------------------------------------------|----------------------------|--------------------------|
| [TenancyManager](./tenancy.md)                     | Tenancy instances          | `multitenancy.tenancies` |
| [TenantProviderManager](./tenant-providers.md)     | TenantProvider instances   | `multitenancy.providers` |
| [IdentityResolverManager](./identity-resolvers.md) | IdentityResolver instances | `multitenancy.resolvers` |
| ServiceOverrideManager                             | ServiceOverride instances  | `multitenancy.overrides` |

The first three extend `BaseFactory` and share the same creation pattern. `ServiceOverrideManager` handles overrides
differently due to their lifecycle requirements.

## BaseFactory

The `BaseFactory` class provides the core factory logic that managers inherit:

```php
abstract class BaseFactory
{
    protected static array $customCreators = [];  // Registered drivers
    protected array $objects = [];                 // Instance cache

    abstract protected function getFactoryName(): string;
    abstract protected function getConfigKey(string $name): string;
}
```

Subclasses implement two methods:

- **`getFactoryName()`** — Returns the type name (e.g., `'tenancy'`, `'provider'`, `'resolver'`)
- **`getConfigKey()`** — Maps a name to its config path (e.g., `'tenants'` → `'multitenancy.providers.tenants'`)

## Driver Resolution

When you request an instance, the factory resolves it through a priority chain:

```
get('subdomain')
    ↓
Check instance cache → return if exists
    ↓
Read config for 'subdomain'
    ↓
Get driver name from config (e.g., 'subdomain')
    ↓
1. Check custom creators → use if registered
2. Look for create{Driver}{Type} method → use if exists
3. Look for createDefault{Type} method → use if no driver specified
4. Throw exception if nothing found
    ↓
Cache and return instance
```

### Method Naming Convention

Built-in drivers are created by conventionally-named methods:

```php
// For driver 'eloquent' in TenantProviderManager:
// Looks for: createEloquentProvider
$method = 'create' . ucfirst($driver) . ucfirst($this->getFactoryName());
```

Examples:

| Manager                 | Driver      | Method                      |
|-------------------------|-------------|-----------------------------|
| TenantProviderManager   | `eloquent`  | `createEloquentProvider()`  |
| TenantProviderManager   | `database`  | `createDatabaseProvider()`  |
| IdentityResolverManager | `subdomain` | `createSubdomainResolver()` |
| IdentityResolverManager | `path`      | `createPathResolver()`      |
| IdentityResolverManager | `header`    | `createHeaderResolver()`    |
| TenancyManager          | (default)   | `createDefaultTenancy()`    |

This convention makes it easy to add new built-in drivers — just add a method following the pattern.

## Registering Custom Drivers

Third-party packages or applications can register custom drivers:

```php
TenantProviderManager::register('api', function (Application $app, array $config, string $name) {
    return new ApiTenantProvider($name, $config['endpoint'], $config['api_key']);
});
```

The closure receives:

- **`$app`** — The Laravel application container
- **`$config`** — The full configuration array for this instance
- **`$name`** — The configuration name (e.g., `'tenants'`)

Once registered, the driver can be used in configuration:

```php
'providers' => [
    'external' => [
        'driver' => 'api',
        'endpoint' => 'https://api.example.com/tenants',
        'api_key' => env('TENANT_API_KEY'),
    ],
],
```

Custom creators take priority over built-in methods, allowing you to override default behaviour.

## Instance Caching

Managers cache instances by name:

```php
public function get(?string $name = null): object
{
    $name ??= $this->getDefaultName();

    if (! isset($this->objects[$name])) {
        $this->objects[$name] = $this->resolve($name);
    }

    return $this->objects[$name];
}
```

This means:

- First call creates and caches the instance
- Subsequent calls return the cached instance
- Each named configuration has its own cached instance
- `flushResolved()` clears the cache if you need fresh instances

## Post-Resolution Setup

After creating an instance, the factory automatically injects common dependencies:

```php
protected function setupResolvedObject(object $object): object
{
    if (method_exists($object, 'setApp')) {
        $object->setApp($this->app);
    }

    if (method_exists($object, 'setSprout')) {
        $object->setSprout($this->app->make(Sprout::class));
    }

    return $object;
}
```

Classes can use the `AwareOfApp` and `AwareOfSprout` traits to receive these injections:

```php
class CustomResolver extends BaseIdentityResolver
{
    use AwareOfApp, AwareOfSprout;

    public function someMethod(): void
    {
        $this->getApp()->make(SomeService::class);
        $this->getSprout()->tenancies()->get();
    }
}
```

## The Sprout Orchestrator

The `Sprout` class composes all four managers and provides access to them:

```php
$sprout = app(Sprout::class);

$sprout->providers();   // TenantProviderManager
$sprout->resolvers();   // IdentityResolverManager
$sprout->tenancies();   // TenancyManager
$sprout->overrides();   // ServiceOverrideManager
```

This is the primary entry point for accessing Sprout's functionality. The service provider registers `Sprout` as a
singleton, and the managers are accessed through it.

## Configuration Structure

The configuration mirrors the manager structure:

```php
return [
    'defaults' => [
        'tenancy'  => 'tenants',    // Default tenancy name
        'provider' => 'tenants',    // Default provider name
        'resolver' => 'subdomain',  // Default resolver name
    ],

    'tenancies' => [
        'tenants' => [
            'provider' => 'tenants',
            'options'  => [...],
        ],
    ],

    'providers' => [
        'tenants' => [
            'driver' => 'eloquent',
            'model'  => Tenant::class,
        ],
    ],

    'resolvers' => [
        'subdomain' => [
            'driver' => 'subdomain',
            'domain' => env('TENANTED_DOMAIN'),
        ],
    ],

    'overrides' => [...],
];
```

Each section defines named configurations. The `driver` key determines which creator method or custom closure handles
instantiation.

## Extending Sprout

To add a custom driver:

1. **Create the implementation** — Implement the appropriate contract (`TenantProvider`, `IdentityResolver`, etc.)
2. **Register the driver** — Call `Manager::register()` in a service provider's `register()` method
3. **Configure it** — Add the configuration with your driver name

```php
// In your service provider
public function register(): void
{
    IdentityResolverManager::register('jwt', function ($app, $config, $name) {
        return new JwtIdentityResolver(
            $name,
            $config['secret'],
            $config['claim'] ?? 'tenant_id'
        );
    });
}
```

```php
// In config/multitenancy.php
'resolvers' => [
    'jwt' => [
        'driver' => 'jwt',
        'secret' => env('JWT_SECRET'),
        'claim' => 'org_id',
    ],
],
```

## ServiceOverrideManager Differences

The `ServiceOverrideManager` doesn't extend `BaseFactory` because overrides have different lifecycle requirements:

- Overrides are **setup and cleaned up** during tenant context changes
- They may be **bootable** (need to run after all overrides are registered)
- They operate on the **service container** rather than being simple instances

See [Service Overrides](./service-overrides.md) for details on how overrides work.

## Related Documents

- [Tenancy](./tenancy.md) — What TenancyManager creates
- [Tenant Providers](./tenant-providers.md) — What TenantProviderManager creates
- [Identity Resolvers](./identity-resolvers.md) — What IdentityResolverManager creates
- [Service Overrides](./service-overrides.md) — How ServiceOverrideManager differs
