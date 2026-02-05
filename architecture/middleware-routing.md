# Middleware & Routing

Sprout provides route macros and middleware that integrate tenant resolution into Laravel's routing system. This
document covers the routing API itself — for when and why resolution happens, see [Resolution Hooks](./resolution-hooks.md).

## Route Macros

Sprout registers two macros on Laravel's router:

```php
Route::tenanted(function () {
    Route::get('/dashboard', DashboardController::class);
}, 'subdomain', 'tenants');

Route::possiblyTenanted(function () {
    Route::get('/about', AboutController::class);
}, 'header', 'tenants');
```

**`Route::tenanted()`** creates routes that require a tenant. Requests without a tenant throw `NoTenantFoundException`.

**`Route::possiblyTenanted()`** creates routes where tenants are optional. Resolution is attempted, but requests proceed
even without a tenant.

Both accept optional resolver and tenancy names. When omitted, they use configured defaults.

### What the Macros Do

These aren't simple route groups with middleware attached. They orchestrate:

1. **Instance resolution** — Get the resolver and [tenancy](./tenancy.md) from their managers
2. **Route configuration** — Call the resolver's `configureRoute()` to apply constraints (domains, prefixes)
3. **Middleware attachment** — Add Sprout's middleware with parameters encoding the resolver and tenancy
4. **Route wrapping** — Execute the closure within the configured group

The `configureRoute()` method lets each [resolver type](./identity-resolvers.md) customize route definitions without
coupling route files to resolver implementation details.

## The configureRoute() Pattern

[Different resolver types](./identity-resolvers.md#two-categories) need different route configurations:

**[Subdomain resolver](./identity-resolvers.md#routing-resolvers)** sets a domain constraint:

```php
$route->domain('{tenants_subdomain}.example.com');
```

**[Path resolver](./identity-resolvers.md#routing-resolvers)** sets a URL prefix:

```php
$route->prefix('{tenants_path}');
```

**[Non-routing resolvers](./identity-resolvers.md#non-routing-resolvers)** (header, cookie, session) don't modify route
matching — they extract identifiers from request metadata, not URL structure. However, the header resolver does add
[response middleware](#header-response-middleware) to echo the resolved tenant identifier back.

This design means you can switch resolvers without changing route definitions. The macro handles the differences.

## Dynamic Parameter Names

Routing resolvers face a naming collision problem. What if two tenancies both use subdomain resolution? Both would need
a `{tenant}` parameter.

Sprout solves this with dynamic parameter names using [placeholders](./identity-resolvers.md#placeholders), following
the pattern `{tenancy}_{resolver}`:

- Default tenancy with subdomain resolver: `{tenants_subdomain}`
- Organizations tenancy with path resolver: `{organizations_path}`

This is transparent to your application. Each resolver provides a `route()` helper that generates URLs with the correct
parameter name — see [URL Generation](./identity-resolvers.md#url-generation) for details.

## Middleware

Two middleware classes handle tenant context:

**`sprout.tenanted`** — For required tenant routes. Triggers resolution (if the middleware hook is active), then
verifies a tenant exists. Throws `NoTenantFoundException` if not.

**`sprout.tenanted.optional`** — For optional tenant routes. Same resolution logic, but allows requests to proceed
without a tenant.

The middleware parameters encode the resolver and tenancy:

```
sprout.tenanted:subdomain,tenants
```

### Dual Purpose

The middleware serves two roles depending on the active [resolution hook](./resolution-hooks.md):

- **Routing hook active:** Middleware is just a marker. Resolution already happened via the `RouteMatched` event
  listener. The middleware only verifies the tenant exists (for required routes).

- **Middleware hook active:** Middleware triggers resolution. It calls `ResolutionHelper::handleResolution()` to
  identify and load the tenant.

If both hooks are enabled, the routing hook resolves first. The middleware sees `tenancy->wasResolved()` is true and
skips resolution.

### Header Response Middleware

The [header resolver](./identity-resolvers.md#non-routing-resolvers) adds `AddTenantHeaderToResponse` middleware via its
`configureRoute()` method. This middleware includes the resolved tenant identifier in response headers, confirming which
tenant handled the request.

This is useful for debugging and for clients that need to verify their tenant header was correctly interpreted. The
header name matches the one configured for the resolver.

## Multiple Tenancies

A route can participate in multiple [tenancies](./tenancy.md):

```php
Route::tenanted(function () {
    Route::tenanted(function () {
        // Both organization AND team context
    }, 'path', 'teams');
}, 'subdomain', 'organizations');
```

Each tenancy resolves independently. The dynamic parameter naming prevents URL collisions.

## Constraints and Gotchas

**Route caching works normally.** The macros generate standard Laravel route definitions.

**Middleware ordering matters.** Sprout's middleware should run after Laravel's session and cookie middleware (if using
those resolvers) but before application middleware that depends on tenant context.

**Error pages may lack tenant context.** If an error occurs before resolution, or on a non-tenanted route, there's no
tenant. Error handlers should check for tenant presence.

**Route model binding timing.** Binding happens after Sprout's resolution. The tenant is already loaded — you don't need
route model binding for the tenant itself.

**Fallback resolution.** Routing resolvers prefer route parameters but fall back to parsing the request directly. This
handles edge cases like error pages outside tenanted route groups.

## Related Documents

- [Identity Resolvers](./identity-resolvers.md) — How resolvers extract tenant identity
- [Resolution Hooks](./resolution-hooks.md) — When and why resolution occurs at different points
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — What happens after resolution
