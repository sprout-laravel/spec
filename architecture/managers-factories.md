# Managers & Factories

Sprout uses a driver-based factory pattern across its core components. This pattern enables third-party packages to
extend Sprout with custom implementations without modifying core code.

## Why This Pattern

Sprout needs to support multiple implementations of key concepts:

- **Tenant providers** — Eloquent, raw database, external APIs
- **Identity resolvers** — Subdomain, path, header, cookie, session, custom
- **Service overrides** — Cache, session, filesystem, and more

Rather than hardcoding these, Sprout uses configuration-driven factories. You specify a driver name in config, and the
factory creates the appropriate implementation. This keeps the core flexible and allows packages like Bud, Seedling, and
Canopy to register their own drivers.

## The BaseFactory Pattern

Three of Sprout's managers extend a common `BaseFactory` class. This isn't just code reuse — it's a deliberate design
decision that ensures consistency across the system.

The base factory provides:

- **Configuration-driven creation** — Read a config section, find the driver, create the instance
- **Instance caching** — Create once, return the same instance on subsequent requests
- **Custom driver registration** — Allow third parties to register drivers via closures
- **Dependency injection** — Automatically inject the app container and Sprout instance into created objects

Subclasses only need to implement two methods:

- `getFactoryName()` — Returns the type name (e.g., `'provider'`, `'resolver'`)
- `getConfigKey()` — Maps a name to its config path

This means adding a new manager (as extension packages do) follows a predictable pattern. You extend `BaseFactory`,
implement the two methods, add your driver creation methods, and the rest works automatically.

## Driver Resolution

When you request an instance (e.g., `$sprout->providers()->get('tenants')`), the factory resolves it through a priority
chain:

1. **Check cache** — If this name was already resolved, return the cached instance
2. **Read configuration** — Get the config for this name, including the `driver` value
3. **Check custom creators** — If a third party registered a creator for this driver, use it
4. **Check convention methods** — Look for a method like `createEloquentProvider()` for driver `eloquent`
5. **Check default method** — If no driver specified, look for `createDefaultProvider()`
6. **Throw exception** — If nothing matches, the configuration is invalid

This priority order is deliberate:

- **Custom creators first** means third parties can override built-in drivers if needed
- **Convention methods** keep built-in driver code organised and discoverable
- **Default methods** handle the case where config omits a driver entirely
- **Caching** ensures the same instance is returned throughout a request

The convention for built-in methods is `create{Driver}{Type}()` — so `createEloquentProvider()`,
`createSubdomainResolver()`, etc. This makes it easy to find how a driver is created and to add new built-in drivers.

## The Four Managers

| Manager | Creates | Config Key |
|---------|---------|------------|
| [TenancyManager](./tenancy.md) | Tenancy instances | `multitenancy.tenancies` |
| [TenantProviderManager](./tenant-providers.md) | TenantProvider implementations | `multitenancy.providers` |
| [IdentityResolverManager](./identity-resolvers.md) | IdentityResolver implementations | `multitenancy.resolvers` |
| [ServiceOverrideManager](./service-overrides.md) | ServiceOverride implementations | `multitenancy.overrides` |

The first three extend `BaseFactory`. `ServiceOverrideManager` is different — overrides have lifecycle requirements
(setup/cleanup phases, bootable overrides) that don't fit the simple factory pattern. See
[Service Overrides](./service-overrides.md) for details on why.

## Registering Custom Drivers

The primary extension point is `register()`. Third-party packages call this to add new drivers:

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

Custom creators take priority over built-in drivers, allowing you to override default behaviour if needed.

## Built-in Driver Resolution

For built-in drivers, the factory uses a naming convention. When you specify `'driver' => 'eloquent'` for a provider,
the factory looks for a method called `createEloquentProvider()`. This convention makes it easy to add new built-in
drivers — just add a method following the pattern.

| Manager | Driver | Method |
|---------|--------|--------|
| TenantProviderManager | `eloquent` | `createEloquentProvider()` |
| TenantProviderManager | `database` | `createDatabaseProvider()` |
| IdentityResolverManager | `subdomain` | `createSubdomainResolver()` |
| IdentityResolverManager | `path` | `createPathResolver()` |

## Accessing Managers

The `Sprout` class provides access to all managers:

```php
$sprout = app(Sprout::class);

$sprout->providers();   // TenantProviderManager
$sprout->resolvers();   // IdentityResolverManager
$sprout->tenancies();   // TenancyManager
$sprout->overrides();   // ServiceOverrideManager
```

From a manager, you can get specific instances:

```php
$provider = $sprout->providers()->get('tenants');
$resolver = $sprout->resolvers()->get('subdomain');
```

## Extension Packages

This factory pattern is how Sprout's extension packages integrate:

- **Bud** registers drivers for tenant-specific configuration
- **Seedling** registers drivers for multi-database support
- **Canopy** registers drivers for domain-based identification

Each package calls the appropriate `Manager::register()` method in its service provider, making its drivers available
through standard configuration.

## Related Documents

- [Tenancy](./tenancy.md) — What TenancyManager creates
- [Tenant Providers](./tenant-providers.md) — What TenantProviderManager creates
- [Identity Resolvers](./identity-resolvers.md) — What IdentityResolverManager creates
- [Service Overrides](./service-overrides.md) — What ServiceOverrideManager creates
