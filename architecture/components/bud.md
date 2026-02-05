# Bud

Bud provides tenant-specific configuration for Laravel's driver-based services. It lets each tenant have their own
cache stores, database connections, filesystem disks, mail transports, broadcast connections, and authentication
providers — all configured at runtime based on who the current tenant is.

## What Bud Solves

Core's [service overrides](../service-overrides.md) make Laravel services tenant-aware by scoping them — adding tenant
prefixes to cache keys, storing files in tenant directories, isolating sessions. But they assume all tenants use the
same underlying infrastructure with the same credentials.

Some applications need more. Each tenant might have:

- Their own Redis server for caching
- Their own database with separate credentials
- Their own S3 bucket for file storage
- Their own SMTP server or Mailgun account

Bud solves this by storing service configuration per-tenant and injecting it at runtime. When a tenant becomes active
and your application requests a cache store, Bud fetches that tenant's cache configuration and creates a store using
their specific credentials.

## Config Stores

Config stores are the storage layer — where tenant-specific configuration lives. A config store holds configuration
entries keyed by four values:

- **Tenancy** — which tenancy this config belongs to
- **Tenant** — which tenant within that tenancy
- **Service** — which Laravel service (cache, database, filesystem, etc.)
- **Name** — the specific connection/store/disk name

This four-part key means the same tenant can have different configurations for different services, and different named
configurations within the same service (e.g., separate cache stores for different purposes).

### Encryption

All configuration is encrypted at rest. Config stores use Laravel's encrypter by default, but you can provide a custom
encryption key per store. This is important because tenant configuration often contains sensitive credentials — database
passwords, API keys, secret tokens.

### Built-in Drivers

**Database** stores configuration in a database table. The table has columns for tenancy, tenant_id, service, name, and
the encrypted config blob. Good for applications that want configuration managed alongside other tenant data, or that
need to query/update configuration programmatically.

**Filesystem** stores configuration as encrypted files on a disk. Files are organised by tenancy and tenant resource
key, with subdirectories for service and name. Good for applications that prefer configuration as files, or that want
to version-control tenant configuration.

The filesystem store requires tenants to implement `TenantHasResources` — it uses the resource key to build the file
path, providing a consistent directory structure that doesn't expose tenant IDs.

## The "bud" Driver

Bud registers a `bud` driver with each supported Laravel service manager. When you configure a store, connection, or
disk with `'driver' => 'bud'`, you're telling Laravel to delegate to Bud for the real configuration.

```php
// config/cache.php
'stores' => [
    'tenant' => [
        'driver' => 'bud',
        'store'  => 'tenant',  // The name to look up in the config store
    ],
],
```

When your application requests this cache store, Bud:

1. Gets the current tenancy and tenant from Sprout
2. Fetches configuration from the config store for service `cache`, name `tenant`
3. Merges that configuration with any static config from the array
4. Creates the real cache store using the merged configuration

The configuration in the store contains everything needed to create the real driver — the actual driver name (redis,
file, database), connection details, credentials. Bud retrieves it, decrypts it, and passes it to Laravel's cache
manager to build the real store.

### Cyclic Detection

What if a tenant's configuration in the store also specifies `'driver' => 'bud'`? That would create an infinite loop.
Bud detects this and throws `CyclicOverrideException` before it happens.

## How It Works

The flow from request to configured service:

```
Application requests cache store "tenant"
         │
         ▼
┌─────────────────────────────────┐
│   Laravel Cache Manager         │
│   Sees driver = "bud"           │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Bud Cache Store Creator       │
│   Gets current tenancy/tenant   │
│   Fetches config from store     │
│   Checks for cyclic driver      │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Config Store                  │
│   Decrypts and returns config   │
│   {driver: "redis", host: ...}  │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Laravel Cache Manager         │
│   Builds real Redis store       │
│   using tenant's config         │
└─────────────────────────────────┘
         │
         ▼
    Cache store ready
```

### Cleanup

Bud's overrides track which stores/connections/disks were created during a tenant's session. When the tenant changes,
cleanup purges these from Laravel's managers so they're recreated with the new tenant's configuration. This prevents
stale connections and ensures each tenant always gets their own configured services.

## Supported Services

Bud provides overrides for six Laravel services. Since Bud simply stores and retrieves configuration arrays, it works
with any driver that Laravel's managers support — including custom drivers registered by other packages.

### Cache

Tenant-specific cache stores. The stored configuration is passed directly to Laravel's cache manager, so any cache
driver works — file, database, Redis, Memcached, DynamoDB, or custom drivers.

### Database

Tenant-specific database connections. Any database driver Laravel supports — MySQL, PostgreSQL, SQLite, SQL Server —
works with tenant-specific credentials and connection details.

### Filesystem

Tenant-specific filesystem disks. Works with local disks, S3, any S3-compatible storage, SFTP, FTP, or custom
filesystem adapters.

### Mail

Tenant-specific mail transports. Works with SMTP, Mailgun, Postmark, SES, or any mailer Laravel supports.

### Broadcast

Tenant-specific broadcast connections. Works with Pusher, Ably, Redis, or custom broadcast drivers.

### Auth

Tenant-specific authentication providers. Works with Eloquent, database, or custom user providers.

## Configuration

### Config Stores

Configure stores in `config/sprout/bud.php`:

```php
'stores' => [
    'database' => [
        'driver'     => 'database',
        'connection' => null,        // Uses default connection
        'table'      => 'tenant_config',
        'key'        => null,        // Uses app key, or provide custom
    ],
    'files' => [
        'driver'    => 'filesystem',
        'disk'      => 'local',
        'directory' => 'tenant-config',
        'key'       => env('TENANT_CONFIG_KEY'),
    ],
],
```

### Tenancy Options

Control which config store a tenancy uses via `BudOptions`:

```php
'tenancies' => [
    'tenants' => [
        'options' => [
            BudOptions::useDefaultStore('database'),
            // or
            BudOptions::alwaysUseStore('database'),  // Locks to this store
        ],
    ],
],
```

The difference: `useDefaultStore` sets the default but allows override per-request. `alwaysUseStore` locks the tenancy
to that store — useful when you need to guarantee configuration comes from a specific source.

### Service Configuration

Configure services to use the `bud` driver:

```php
// config/cache.php
'stores' => [
    'tenant' => [
        'driver'   => 'bud',
        'store'    => 'tenant',
        'budStore' => 'database',  // Optional: specify which config store
    ],
],
```

The `budStore` option lets you specify which config store to use for this particular service configuration, overriding
the tenancy default.

## Contextual Attributes

Bud provides PHP 8 attributes for dependency injection:

```php
public function __construct(
    #[ConfigStore('database')] ConfigStoreContract $store,
    #[TenantConfig('cache', 'main')] ?array $cacheConfig,
) {
    // $store is the 'database' config store instance
    // $cacheConfig is the current tenant's cache.main configuration
}
```

These use Laravel's contextual attribute system to inject Bud resources directly into your classes.

## Constraints and Gotchas

**Requires tenant context.** The `bud` driver only works within an active tenant context. If code requests a
bud-configured service outside of tenant context, Bud throws `TenancyMissingException` or `TenantMissingException`.

**Filesystem store requires TenantHasResources.** The filesystem config store builds paths using the tenant's resource
key. If your tenant model doesn't implement `TenantHasResources`, you'll get a `MisconfigurationException`.

**Configuration must exist.** If a tenant doesn't have configuration for a requested service/name combination, Bud
throws a runtime exception. There's no automatic fallback to a default configuration — the tenant must have their
config stored.

**Encryption keys matter.** If you use custom encryption keys for config stores, losing the key means losing access to
all stored configuration. Treat config store encryption keys with the same care as your application key.

**No hot-reloading.** Configuration is fetched when a service is first requested. If you update a tenant's
configuration mid-request, already-created services won't see the change until the next request (or until cleanup runs
on tenant switch).

## Related Documents

- [Service Overrides](../service-overrides.md) — Core's approach to tenant-aware services
- [Tenancy](../tenancy.md) — How tenancy options work
- [Managers & Factories](../managers-factories.md) — The factory pattern Bud's ConfigStoreManager extends
