# Resolution Hooks

Resolution hooks are points in Laravel's request lifecycle where tenant identification can occur. Sprout provides two
hooks: one during routing, one during middleware execution. The choice of hook affects when tenant context becomes
available and which resolvers can be used.

## The Two Hooks

### Routing Hook

The routing hook fires when Laravel's `RouteMatched` event occurs — after the router has matched the request to a route
but before middleware begins executing.

At this point:

- The matched route is available (including route parameters)
- Sessions are NOT available (session middleware hasn't run)
- Auth is NOT available (auth middleware hasn't run)
- Service containers bindings from middleware aren't available yet

This is the earliest possible moment for tenant resolution. If you resolve here, tenant context is available throughout
the entire middleware stack. [Service overrides](./service-overrides.md) can configure services before any middleware uses them.

Most [resolvers](./identity-resolvers.md) work at the routing hook: subdomain, path, header, and cookie. They extract identifiers from URLs or
request metadata, none of which requires middleware to have run.

### Middleware Hook

The middleware hook fires during middleware stack execution, specifically when Sprout's tenant middleware runs.

At this point:

- Sessions ARE available (if session middleware ran earlier)
- Auth may be available (if auth middleware ran earlier)
- Other middleware-provided context is available

This hook exists primarily for the [session resolver](./identity-resolvers.md#session), which needs session data to determine the tenant. It's also useful
as a fallback — if the routing hook didn't resolve a tenant (perhaps the resolver couldn't run at that hook), the
middleware hook gets another chance.

## Why Both Exist

Different resolvers have different requirements.

The **session resolver** stores tenant identity in the session. Sessions don't exist during routing — Laravel's session
middleware creates them. So the session resolver can only work at the middleware hook.

Other resolvers (subdomain, path, header, cookie) work at both hooks. For these, the routing hook is preferable because
it's earlier — tenant context is available sooner, and service overrides can configure services before any middleware
runs.

Having both hooks means you get the best of both worlds:

- Early resolution when possible (routing hook)
- Session-based resolution when needed (middleware hook)
- Fallback if early resolution fails

## How the Hooks Interact

The hooks don't compete — they cooperate.

If a tenant is resolved at the routing hook, the middleware hook skips resolution. The [tenancy](./tenancy.md) already has a tenant;
there's nothing more to do. The middleware still runs, but only to enforce that a tenant exists (for required routes).

If the routing hook doesn't resolve a tenant (resolver couldn't run, identifier not found, hook disabled), the
middleware hook attempts resolution. This is the fallback path.

This interaction happens automatically. Resolvers declare which hooks they support via `canResolve()`, and Sprout checks
before attempting resolution.

## Required vs Optional Tenants

Resolution hooks determine *when* identification happens. Middleware determines *whether* a tenant is required.

**Required tenant routes** use `sprout.tenanted` middleware. After all hooks have run, if there's no tenant, the
middleware throws `NoTenantFoundException`. The route requires a tenant; missing one is an error.

**Optional tenant routes** use `sprout.tenanted.optional` middleware. If no tenant is found after all hooks, that's
fine. The route works with or without tenant context.

The distinction is about enforcement, not resolution. Both middleware types attempt resolution at the middleware hook
(if enabled). The difference is what happens when resolution fails.

## Configuration

Hooks are enabled in configuration:

```php
// config/sprout/core.php
'hooks' => [
    ResolutionHook::Routing,
    ResolutionHook::Middleware,
],
```

By default, both hooks are enabled. The order in the array doesn't matter — each hook fires at its predetermined point
in the lifecycle.

### Disabling Hooks

You might disable a hook for simplicity or performance:

**Disable Routing hook** if you only use the session resolver. Since it can't run at the routing hook anyway, there's
no point having the hook enabled. But note that disabling means no early resolution — tenant context won't be available
until middleware runs.

**Disable Middleware hook** if you only use resolvers that work at the routing hook and don't need the fallback. This
is rare — most applications benefit from having both enabled.

## Middleware Parameters

The tenant middleware accepts optional parameters to specify which resolver and tenancy to use:

```php
// Default resolver and tenancy
Route::middleware('sprout.tenanted')->group(...);

// Specific resolver
Route::middleware('sprout.tenanted:subdomain')->group(...);

// Specific resolver and tenancy
Route::middleware('sprout.tenanted:subdomain,organizations')->group(...);
```

The `Route::tenanted()` macro handles this for you, but understanding the format helps when debugging or configuring
manually.

## Middleware Priority

When using the session resolver, Sprout's middleware must run *after* Laravel's session middleware. Otherwise, sessions
won't be available when resolution is attempted.

Laravel's middleware priority system handles this. If you encounter issues, you can explicitly set the priority:

```php
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->appendToPriorityList(
        StartSession::class,
        SproutTenantContextMiddleware::class
    );
})
```

Only adjust priority when using session-dependent resolvers. For most applications with subdomain, path, or header
resolvers, the default priority works fine.

## Lifecycle Position

```
Request arrives
      │
      ▼
Route matching
      │
      ▼
RouteMatched event
   ┌──┴───────────┐
   │ Routing Hook │  ← Tenant identified here if possible
   └──┬───────────┘
      │
      ▼
Middleware stack begins
      │
      ▼
Session middleware (if present)
      │
      ▼
Sprout tenant middleware
   ┌──┴──────────────┐
   │ Middleware Hook │  ← Fallback resolution, enforcement
   └──┬──────────────┘
      │
      ▼
Remaining middleware
      │
      ▼
Controller
```

The earlier resolution happens, the more of the request has tenant context available.

## Constraints and Gotchas

**The Booting hook exists but isn't implemented.** The `ResolutionHook` enum includes a `Booting` case, reserved for
potential future use. Currently, only `Routing` and `Middleware` are functional.

**Session resolver only works at Middleware hook.** It explicitly checks the hook and refuses to resolve at Routing.
This isn't a limitation to work around — it's fundamental to how sessions work.

**Disabling Routing hook affects service overrides.** If overrides run before tenant context exists, they can't
configure services correctly. This matters for overrides like cache and filesystem that need tenant information during
setup.

**The routing hook listener checks for Sprout middleware.** Resolution at the routing hook only happens if the matched
route has Sprout's tenant middleware. Routes without the middleware don't trigger resolution, even if the hook is
enabled.

## Related Documents

- [Identity Resolvers](./identity-resolvers.md) — How resolvers extract identifiers and which hooks they support
- [Tenancy](./tenancy.md) — How resolved tenants are stored and tracked
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events triggered after resolution
- [Service Overrides](./service-overrides.md) — Why early resolution matters for overrides
