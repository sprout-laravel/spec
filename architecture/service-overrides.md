# Service Overrides

Service overrides are hooks into the [tenancy lifecycle](./tenancy-lifecycle.md). When a tenant becomes active, overrides perform setup actions.
When the tenant is deactivated, they perform cleanup. The name "service override" reflects their primary use case —
making Laravel services tenant-aware — but they're not limited to services. Any action that should respond to tenant
context changes can be implemented as an override.

## Why Overrides Exist

Laravel services are designed for single-tenant applications. The cache stores data. The filesystem writes files. The
session tracks user state. None of them know about tenants.

In a multitenant application, these services need to be tenant-aware:

- Cache keys should be isolated per tenant
- Files should be stored in tenant-specific directories
- Sessions shouldn't leak between tenants
- Auth guards should reset when the tenant changes

You could manually handle this everywhere you use these services. But that's error-prone and scattered throughout your
codebase. Overrides centralise this logic — when the tenant changes, the override handles making the service aware.

## The Two-Phase Pattern

Overrides have two phases: **setup** and **cleanup**.

**Setup** runs when a tenant becomes active. This is where you configure services for the new tenant — set cache
prefixes, configure filesystem paths, update session settings.

**Cleanup** runs when a tenant is deactivated (either being replaced or leaving tenant context entirely). This is where
you undo what setup did — clear cached references, reset configurations, forget resolved instances.

When switching tenants (A → B), cleanup runs for A, then setup runs for B. The order matters — you clean up the old
context before establishing the new one.

## Bootable Overrides

Some overrides need to modify services before any tenant is active. They need to register custom drivers, extend
managers, or replace container bindings. This happens during Laravel's boot phase, before any request processing.

These are **bootable overrides**. They have a `boot()` method that runs once during application startup. After that,
`setup()` and `cleanup()` work as normal.

The distinction matters because:

- Boot happens once, setup/cleanup happen per tenant change
- Boot has access to the application container during startup
- Boot is where you extend Laravel's managers with custom drivers

For example, the cache override registers a `sprout` driver during boot. That driver exists for the entire application
lifetime. When tenants change, setup/cleanup configure which tenant the driver operates on.

## Stacked Overrides

Some services need multiple modifications that are logically separate. The filesystem needs both a custom manager and a
custom driver. Auth needs both guard resetting and password broker replacement.

Rather than cramming everything into one override, Sprout supports **stacked overrides** — a composite override
containing multiple sub-overrides. Each sub-override handles one concern. The stack coordinates them.

When you enable `filesystem`, you're actually enabling a stack containing:

1. **FilesystemManagerOverride** — Replaces Laravel's filesystem manager with one that supports tenant context
2. **FilesystemOverride** — Registers the `sprout` driver for tenant-scoped disks

Both run during boot, both run during setup/cleanup, but each handles its own responsibility.

## Built-in Overrides

Sprout includes overrides for Laravel's core services:

### cache

Registers a `sprout` cache driver. Configure stores with `'driver' => 'sprout'` and they automatically scope to the
current tenant. The underlying driver (redis, file, database) handles actual storage; the sprout wrapper adds tenant
isolation.

On cleanup, tracked stores are forgotten so they're recreated with the new tenant's prefix.

### session

Extends file and native session handlers to store sessions in tenant-specific locations. Optionally extends the database
handler too.

On setup, reconfigures session settings (path, domain, secure, same_site) from tenant context and sets a tenant-specific
cookie name. On cleanup, restores original settings.

### cookie

Configures cookie defaults (path, domain, secure, same_site) based on tenant context. Not bootable — just updates
settings on tenant change.

### filesystem

A stacked override. The manager override replaces Laravel's filesystem manager. The driver override registers a `sprout`
disk driver. Configure disks with `'driver' => 'sprout'` and they automatically use tenant-specific root paths.

On cleanup, tracked disks are forgotten.

### job

Registers a listener for `JobProcessing`. When a queued job runs, the listener restores tenant context from Laravel's
Context facade (which was populated when the job was dispatched). This enables tenant context to survive the
queue/worker boundary.

No setup/cleanup needed — the listener handles everything.

### auth

A stacked override. The guard override forgets resolved auth guards on tenant change (so they're re-resolved with new
tenant context). The password override replaces the password broker manager with a tenant-aware version.

## Enabling Overrides

Overrides are registered globally but enabled per-[tenancy](./tenancy.md). Just because an override exists doesn't mean it runs — the
tenancy must opt in.

Enable all overrides:

```php
'tenancies' => [
    'tenants' => [
        'options' => [
            TenancyOptions::allOverrides(),
        ],
    ],
],
```

Enable specific overrides:

```php
'options' => [
    TenancyOptions::overrides(['cache', 'filesystem', 'session']),
],
```

This per-tenancy control matters when you have multiple tenancies with different needs. An API tenancy might only need
cache isolation. A web tenancy might need everything.

## The Override Lifecycle

### Registration (Application Boot)

When the application boots, overrides are created from configuration. Each configured service gets an override instance.
Bootable overrides are noted for the boot phase.

After each override is registered, a `ServiceOverrideRegistered` event is dispatched. This allows other parts of the
system to react to override registration — useful for packages that need to know which overrides are active.

### Boot (Application Booted)

After all service providers finish, bootable overrides run their `boot()` method. This is where they extend managers,
register drivers, and replace bindings.

After each bootable override completes its boot phase, a `ServiceOverrideBooted` event is dispatched. Like the
registration event, this allows other code to react to the override being ready.

### Setup (Tenant Activated)

When a tenant becomes active, enabled overrides run `setup()`. The manager tracks which overrides were set up for each
tenancy.

### Cleanup (Tenant Deactivated)

When a tenant is deactivated, the manager runs `cleanup()` on overrides that were set up for that tenancy. If an
override was set up but is no longer enabled (configuration changed mid-request), an exception is thrown — this
indicates a misconfiguration.

## Configuration

Overrides are configured in `config/sprout/overrides.php`:

```php
return [
    'cache' => [
        'driver' => CacheOverride::class,
    ],
    'session' => [
        'driver'   => SessionOverride::class,
        'database' => false,  // passed to constructor
    ],
    'filesystem' => [
        'driver'    => StackedOverride::class,
        'overrides' => [
            FilesystemManagerOverride::class,
            FilesystemOverride::class,
        ],
    ],
];
```

The `driver` key specifies which class to instantiate. Additional keys become the config array passed to the
constructor.

For stacked overrides, the `overrides` key lists sub-overrides. Each can be a class name or an array with its own
`driver` and config.

## Creating Custom Overrides

Custom overrides let you make any service or system tenant-aware.

### Simple Override

Extend `BaseOverride` and implement setup/cleanup:

```php
class NotificationOverride extends BaseOverride
{
    public function setup(Tenancy $tenancy, Tenant $tenant): void
    {
        // Configure notifications for tenant
        config(['services.mailgun.domain' => $tenant->mail_domain]);
    }

    public function cleanup(Tenancy $tenancy, Tenant $tenant): void
    {
        // Reset to default
        config(['services.mailgun.domain' => config('services.mailgun.default_domain')]);
    }
}
```

### Bootable Override

Implement `BootableServiceOverride` for overrides that need to run at boot:

```php
class QueueOverride extends BaseOverride implements BootableServiceOverride
{
    public function boot(Application $app, Sprout $sprout): void
    {
        // Register tenant-aware queue connector
        $app['queue']->extend('tenant', function () {
            return new TenantQueueConnector();
        });
    }
}
```

### Registration

Add to `config/sprout/overrides.php`:

```php
'notifications' => [
    'driver' => NotificationOverride::class,
],
```

And enable in your tenancy:

```php
TenancyOptions::overrides(['notifications', 'cache', ...]),
```

## Constraints and Gotchas

**Overrides must be idempotent.** Setup might run multiple times if tenants change during a request. Cleanup might run
without a corresponding setup if configuration changes. Handle these gracefully.

**Boot runs once, not per-tenant.** Don't put tenant-specific logic in boot — that belongs in setup. Boot is for
framework-level configuration that applies regardless of tenant.

**Order matters for stacked overrides.** Sub-overrides run in the order they're listed. If one depends on another's
work, list the dependency first.

**Cleanup receives the departing tenant.** When cleanup runs, `$tenant` is the tenant being deactivated, not the new
one. The new tenant (if any) isn't set yet.

**The manager tracks setup state.** If you bypass the manager and call setup/cleanup directly, the tracking won't match
reality. Always go through the manager.

## Related Documents

- [Tenancy](./tenancy.md) — How tenancies enable overrides via options
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — When overrides are triggered
- [Tenant Providers](./tenant-providers.md) — How tenants are loaded before overrides run
- [Identity Resolvers](./identity-resolvers.md) — Resolver/override conflicts
