# Resolution Hooks

Resolution hooks define the points in Laravel's request lifecycle where tenant identification can occur. Each hook
represents a specific moment when Sprout can attempt to resolve the current tenant.

## Overview

Sprout supports multiple resolution hooks to accommodate different identification strategies. Some strategies (like
subdomain or path) can resolve very early, while others (like session) require Laravel services that aren't available
until later in the lifecycle.

```
Request
   │
   ▼
┌─────────────────────────┐
│    Global Middleware    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│     Route Matching      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Routing Hook          │  ← Route and parameters available
│   (if enabled)          │    Best for most resolvers
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Route Middleware      │
│   ┌───────────────────┐ │
│   │ Middleware Hook   │ │  ← Sessions, auth available
│   │ (if enabled)      │ │    Required for session resolver
│   └───────────────────┘ │
│                         │
│   Tenant required here  │  ← Exception if no tenant
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Controller/Action    │
└─────────────────────────┘
```

## Available Hooks

### Booting

```php
ResolutionHook::Booting
```

Occurs during the booting of service providers.

> **Note:** This hook exists in the enum but is not currently used anywhere in the framework. It's reserved for
> potential future use cases where tenant resolution needs to happen before routing.

### Routing

```php
ResolutionHook::Routing
```

Occurs immediately after the route is matched, before the route middleware stack runs.

**When it fires:**

- After global middleware has run
- After the router has matched the request to a route
- Before route-specific middleware runs

**What's available:**

- The `Route` object and its parameters
- Request attributes set by global middleware
- Basic Laravel services (config, cache, etc.)

**What's NOT available:**

- Session data (session middleware hasn't run)
- Authenticated user (auth middleware hasn't run)
- Anything set by route middleware

**Best for:**

- Subdomain resolution
- Path resolution
- Header resolution
- Cookie resolution
- Domain resolution (Canopy)

This is the recommended hook for most applications because:

1. The tenant is available for dependency injection in middleware constructors
2. Service overrides can set up before other middleware runs
3. Session and auth middleware will have tenant context when they run

### Middleware

```php
ResolutionHook::Middleware
```

Occurs during the route middleware stack, as part of Sprout's tenanted middleware.

**When it fires:**

- During the middleware stack execution
- Position depends on middleware priority configuration

**What's available:**

- Everything from the Routing hook
- Session data (if session middleware has run first)
- Potentially the authenticated user (if auth middleware has run first)

**What's NOT available:**

- Depends on middleware ordering

**Best for:**

- Session resolution (requires session data)
- Any resolver that depends on middleware-provided data

**Important:** This is also where Sprout enforces tenant requirements. If no tenant has been resolved by the time this
hook completes (whether at this hook or an earlier one), an exception is thrown for tenanted routes.

## Configuration

Hooks are enabled in the Sprout configuration:

```php
// config/sprout/core.php
'hooks' => [
    \Sprout\Core\Support\ResolutionHook::Routing,
    \Sprout\Core\Support\ResolutionHook::Middleware,
],
```

The order hooks appear in the configuration array does not matter — each hook fires at its predetermined point in the
lifecycle. The configuration simply controls which hooks are enabled.

**Default configuration:** Both `Routing` and `Middleware` are enabled by default, which covers most use cases.

## How Resolution Works

At each enabled hook point:

1. Sprout checks if the hook is enabled via `Sprout::supportsHook($hook)`
2. If enabled, the identity resolver's `canResolve()` method is called
3. If the resolver can resolve at this hook, resolution is attempted
4. If successful, the tenancy records:
    - The tenant via `setTenant()`
    - The resolver via `resolvedVia()`
    - The hook via `resolvedAt()`

```php
// Resolver decides if it can work at this hook
public function canResolve(Request $request, Tenancy $tenancy, ResolutionHook $hook): bool
{
    // Session resolver can only work during Middleware hook
    if ($this->driver === 'session') {
        return $hook === ResolutionHook::Middleware;
    }
    
    return true;
}
```

## Resolver Compatibility

Not all resolvers work at all hooks:

| Resolver        | Routing | Middleware | Notes                       |
|-----------------|---------|------------|-----------------------------|
| Subdomain       | ✓       | ✓          | Works at either             |
| Path            | ✓       | ✓          | Works at either             |
| Header          | ✓       | ✓          | Works at either             |
| Cookie          | ✓       | ✓          | Works at either             |
| Session         | ✗       | ✓          | Requires session middleware |
| Domain (Canopy) | ✓       | ✓          | Works at either             |

## Middleware Priority

When using the `Middleware` hook with the session resolver, you may need to adjust middleware priority to ensure
Sprout's middleware runs after `StartSession`:

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToPriorityList(
            \Illuminate\Session\Middleware\StartSession::class,
            \Sprout\Http\Middleware\TenantedMiddleware::class
        );
    });
```

> **Warning:** Only adjust middleware priority if you're using the session resolver. Changing priority when using other
> resolvers can cause issues with the session service override and other Sprout features.

## Tracking Resolution

The tenancy tracks where resolution occurred:

```php
// Get the hook where the tenant was resolved
$hook = $tenancy->hook(); // ResolutionHook::Routing

// Check if resolution happened
if ($tenancy->wasResolved()) {
    $resolver = $tenancy->resolver();
    $hook = $tenancy->hook();
}
```

Sprout also tracks the current hook during request processing:

```php
// During request lifecycle
$currentHook = sprout()->getCurrentHook();

// Check if we're at a specific hook
if (sprout()->isCurrentHook(ResolutionHook::Routing)) {
    // ...
}
```

## Common Patterns

### Standard Web Application

Most applications should use the default configuration:

```php
'hooks' => [
    ResolutionHook::Routing,
    ResolutionHook::Middleware,
],
```

With a resolver that works at `Routing` (subdomain, path, header, etc.), the tenant is resolved early and available
throughout the middleware stack.

### Session-Based Identification

For applications using session-based tenant identification:

```php
'hooks' => [
    ResolutionHook::Middleware,
],
```

Or keep both enabled — the session resolver will simply skip the `Routing` hook via its `canResolve()` method.

### API with Multiple Identification Methods

For APIs that accept tenant identification via header or cookie:

```php
'hooks' => [
    ResolutionHook::Routing,
],
```

The `Middleware` hook isn't needed since neither resolver requires session data.
