# Tenancy Lifecycle

The tenancy lifecycle is the sequence of events that occur when a tenant becomes active, changes, or is deactivated.
Understanding this lifecycle is essential for knowing when tenant context is available, when services are reconfigured,
and how tenant state propagates to queued jobs.

## The Central Event

Everything in the lifecycle flows from one event: `CurrentTenantChanged`.

When you call `$tenancy->setTenant($tenant)`, Sprout compares the new tenant to the previous one. If they differ — even
if both are null, or one is null — the event dispatches. This is the trigger for all lifecycle actions. See
[Tenancy](./tenancy.md) for details on the tenancy object itself.

The event carries three pieces of information:

- The tenancy whose tenant changed
- The previous tenant (or null if there wasn't one)
- The current tenant (or null if leaving tenant context)

Every listener receives all three, letting it decide what cleanup or setup is appropriate.

## Why Events Drive the Lifecycle

Sprout could have hardcoded the lifecycle steps directly into `setTenant()`. Instead, it uses events and configurable
listeners. This design choice has several benefits:

**Ordering is explicit.** The bootstrapper list in configuration defines exactly what runs and in what order. You can
see the sequence without reading implementation code.

**Extension is clean.** Adding custom lifecycle actions means adding a listener to the configuration. No need to extend
classes or override methods.

**Removal is possible.** If a built-in bootstrapper doesn't suit your needs, remove it from the configuration. Replace
it with your own, or omit it entirely.

**Testing is simpler.** You can test lifecycle components in isolation because they're separate listeners, not tangled
in a monolithic method.

## The Bootstrapper Sequence

Bootstrappers are event listeners that run when the tenant changes. The term "bootstrapper" emphasises their role —
they bootstrap the application into the correct state for the new tenant context.

The default sequence:

1. **SetCurrentTenantContext** — Stores tenant information in Laravel's Context service
2. **PerformIdentityResolverSetup** — Runs resolver-specific setup (URL defaults, cookies, session storage)
3. **CleanupServiceOverrides** — Reverts the previous tenant's service configuration
4. **SetupServiceOverrides** — Applies the new tenant's service configuration
5. **RefreshTenantAwareDependencies** — Updates container bindings for tenant-aware classes

This order is deliberate. Context storage happens first because other systems might need to read it. Cleanup happens
before setup because you must revert old configuration before applying new. Container refresh happens last because
service overrides need to complete first.

### Context Storage

The first bootstrapper stores the tenant's key in Laravel's Context service. Context is Laravel's mechanism for
propagating request-scoped data to queued jobs.

When a job is dispatched, Laravel automatically captures Context data. When the job runs, that data is restored. By
storing the tenant key in Context, Sprout ensures jobs can restore tenant context without any special handling in your
job classes.

The storage uses tenant keys (not identifiers) because keys are stable internal values. An identifier might change if a
tenant renames their subdomain; the key remains constant. See [Tenant Providers](./tenant-providers.md#the-dual-identity-system) for details on the dual identity system.

### Resolver Setup

Different [resolvers](./identity-resolvers.md) need different post-identification actions. The subdomain resolver registers URL defaults so
generated links automatically include the tenant. The cookie resolver queues a cookie to persist the selection. The
session resolver stores the identifier in session data.

Rather than hardcoding these into the lifecycle, Sprout calls the resolver's `setup()` method. Each resolver handles its
own requirements. If no resolver was involved (direct tenant loading), this step does nothing.

### Service Override Cleanup and Setup

[Service overrides](./service-overrides.md) make Laravel services tenant-aware — scoping cache keys, changing filesystem paths, isolating
sessions. When the tenant changes, the old tenant's configuration must be cleaned up before the new tenant's is applied.

Cleanup runs even when transitioning to no tenant (setting null). This ensures services return to their default,
non-tenant-aware state.

Setup only runs when there's a current tenant. If you're leaving tenant context entirely, there's nothing to set up —
cleanup is sufficient.

See [Service Overrides](./service-overrides.md) for details on what cleanup and setup involve.

### Container Refresh

Some classes implement `TenantAware`, indicating they need to know when the tenant changes. The final bootstrapper
triggers Laravel's container to refresh these dependencies, calling their `setTenant()` method with the new tenant.

This is useful for classes that cache tenant-specific data and need to invalidate it, or services that need to
reconfigure themselves based on the current tenant.

## Two Paths to Tenant Activation

There are two ways to activate a tenant, and the path matters.

### Identification

`identify()` takes a public identifier — the string extracted from a subdomain, path, header, or other source — and
uses the [provider](./tenant-providers.md) to find the matching tenant.

```php
$tenancy->identify('acme');  // Find tenant with identifier 'acme'
```

This dispatches `TenantIdentified` after the tenant is set. Listeners for this event know the tenant came from request
resolution — the identifier was extracted from some external source.

### Loading

`load()` takes an internal key — typically a database primary key — and retrieves the tenant directly.

```php
$tenancy->load(42);  // Find tenant with key 42
```

This dispatches `TenantLoaded` instead. Listeners know the tenant came from internal retrieval — a stored reference,
a job payload, or programmatic loading.

### Why the Distinction Matters

The distinction enables different handling for different contexts.

During request handling, tenants are identified. The identifier came from a URL or header, went through a resolver, and
the full resolution context is available (which resolver, which hook).

During job processing, tenants are loaded. The key was stored in Context when the job was dispatched, and now it's being
used to restore context. There's no resolver involved, no hook — just direct retrieval.

Both paths end up calling `setTenant()`, which dispatches `CurrentTenantChanged` and triggers the bootstrapper sequence.
The `TenantIdentified` and `TenantLoaded` events provide additional context about how the tenant was found.

## Job Context Propagation

When a queued job runs in a worker process, it has no inherent knowledge of which tenant dispatched it. Sprout solves
this through Context propagation:

1. When the tenant is set during the original request, `SetCurrentTenantContext` stores the tenant key in Laravel's Context
2. When a job is dispatched, Laravel captures Context automatically
3. When the job runs, Laravel restores Context before job execution
4. The [job service override](./service-overrides.md#job) listens for `JobProcessing` and calls `$tenancy->load($key)` for each stored tenant

This happens transparently. Your job classes don't need any special code — they run with the same tenant context as the
request that dispatched them.

The override stores keys in a `sprout.tenants` array keyed by tenancy name. This supports multiple tenancies — if both
an organisation and workspace were active when the job was dispatched, both are restored when it runs.

## Switching Tenants

Mid-request tenant switches are possible but rare. When they occur, the full lifecycle runs:

1. `CurrentTenantChanged` dispatches with old as `previous`, new as `current`
2. Cleanup runs for the old tenant
3. Setup runs for the new tenant

The cleanup phase receives the previous tenant, so it knows what to revert. The setup phase receives the current tenant,
so it knows what to configure.

Service overrides must be idempotent — setup might run multiple times if tenants switch repeatedly. The system tracks
which overrides were set up for each tenancy to ensure cleanup matches setup.

## Leaving Tenant Context

Setting the tenant to null is a valid operation:

```php
$tenancy->setTenant(null);  // Leave tenant context
```

This triggers `CurrentTenantChanged` with `current` as null. The bootstrapper sequence runs:

- Context storage removes the tenant entry
- Resolver setup runs with null (allowing resolvers to clean up, like expiring cookies)
- Service override cleanup runs (reverting to default configuration)
- Service override setup is skipped (no tenant to set up)
- Container refresh is skipped (no tenant to inject)

The resolver and hook are also cleared — without a tenant, the resolution context is no longer meaningful.

## Request Lifecycle Integration

The tenancy lifecycle integrates with Laravel's request lifecycle at specific points.

**During routing** (if the Routing hook is enabled), the `RouteMatched` event triggers resolution. The tenant is
identified before middleware runs, so tenant context is available throughout the middleware stack.

**During middleware** (if the Middleware hook is enabled), Sprout's tenant middleware triggers resolution. This is later
in the request lifecycle but allows session-based resolution.

**At request termination**, Sprout resets all tenancies. Each active tenancy's tenant is set to null, triggering cleanup.
This ensures the next request starts fresh, even if the worker handles multiple requests (Octane, Swoole).

See [Resolution Hooks](./resolution-hooks.md) for details on when resolution occurs.

## Configuration

Bootstrappers are configured in `config/sprout/core.php`:

```php
'bootstrappers' => [
    \Sprout\Core\Listeners\SetCurrentTenantContext::class,
    \Sprout\Core\Listeners\PerformIdentityResolverSetup::class,
    \Sprout\Core\Listeners\CleanupServiceOverrides::class,
    \Sprout\Core\Listeners\SetupServiceOverrides::class,
    \Sprout\Core\Listeners\RefreshTenantAwareDependencies::class,
],
```

The order in this array is the execution order. Add custom bootstrappers by appending to the list. Remove built-in ones
by filtering the array. Reorder if your application has specific sequencing needs.

Custom bootstrappers are standard Laravel event listeners:

```php
class MyCustomBootstrapper
{
    public function handle(CurrentTenantChanged $event): void
    {
        if ($event->current !== null) {
            // Set up for new tenant
        }

        if ($event->previous !== null) {
            // Clean up from previous tenant
        }
    }
}
```

## Constraints and Gotchas

**Event discovery can break ordering.** If you create a custom bootstrapper and Laravel's event discovery is enabled,
Laravel may automatically register it as a listener — in addition to the explicit registration from the bootstrapper
configuration. This causes your bootstrapper to run twice, potentially at different points in the sequence. Either
disable event discovery for bootstrapper classes or ensure they're idempotent and handle duplicate execution gracefully.

**Bootstrappers run on every tenant change.** If you switch tenants multiple times in a request, bootstrappers run each
time. Design custom bootstrappers to handle this gracefully.

**The order matters.** Cleanup must happen before setup. If you add a custom bootstrapper, consider where in the sequence
it belongs. Does it depend on service overrides being complete? Put it after setup. Does it need to run before anything
else? Put it first.

**Context only propagates to jobs.** Laravel's Context service specifically supports job propagation. It doesn't
automatically propagate to broadcast events, notifications sent later, or other deferred operations. If you need tenant
context in those places, handle it explicitly.

**Null tenant triggers the lifecycle.** Setting tenant to null isn't a no-op — it triggers `CurrentTenantChanged` and
runs bootstrappers. This is intentional; cleanup needs to happen when leaving tenant context.

**The tenancy stack persists until reset.** Even after setting a tenant to null, the tenancy remains in Sprout's current
tenancy stack. The stack is cleared explicitly at request termination or via `resetTenancies()`.

## Related Documents

- [Tenancy](./tenancy.md) — The container for tenant state
- [Resolution Hooks](./resolution-hooks.md) — When resolution triggers the lifecycle
- [Service Overrides](./service-overrides.md) — What cleanup and setup involve
- [Identity Resolvers](./identity-resolvers.md) — Resolver setup actions
