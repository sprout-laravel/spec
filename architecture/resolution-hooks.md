# Resolution Hooks

Resolution hooks define the points in Laravel's request lifecycle where tenant identification can occur.

## Key Concepts

| Term            | Description                                                               |
|-----------------|---------------------------------------------------------------------------|
| Resolution Hook | A specific point in the Laravel request lifecycle where resolution occurs |
| Handler         | The class responsible for triggering resolution at a specific hook        |
| Hook Support    | Whether a hook is enabled in configuration                                |
| Current Hook    | The hook currently being processed, tracked by Sprout                     |

## The ResolutionHook Enum

**Location:** `Sprout\Core\Support\ResolutionHook`

The `ResolutionHook` enum defines the available points in the Laravel request lifecycle where tenant resolution can be
triggered.

```php
enum ResolutionHook
{
    case Booting;
    case Routing;
    case Middleware;
}
```

### Enum Cases

| Case         | Description                       | Handler Class                   | Status   |
|--------------|-----------------------------------|---------------------------------|----------|
| `Booting`    | During service provider booting   | —                               | Reserved |
| `Routing`    | When `RouteMatched` event fires   | `IdentifyTenantOnRouting`       | Active   |
| `Middleware` | During middleware stack execution | `SproutTenantContextMiddleware` | Active   |

The `Booting` case exists in the enum but is not currently implemented. It is reserved for potential future use cases
where tenant resolution needs to occur before routing.

## Request Lifecycle Position

```
Request
   │
   ▼
┌─────────────────────────┐
│     Route Matching      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   RouteMatched Event    │
│   ┌───────────────────┐ │
│   │   Routing Hook    │ │  ← Route and parameters available
│   └───────────────────┘ │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│      Middleware         │
│   ┌───────────────────┐ │
│   │ Middleware Hook   │ │  ← Sessions, auth available
│   └───────────────────┘ │
│                         │
│   Tenant enforcement    │  ← Exception if no tenant (required middleware)
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Controller/Action    │
└─────────────────────────┘
```

## Handler Classes

### IdentifyTenantOnRouting

**Location:** `Sprout\Core\Listeners\IdentifyTenantOnRouting`

An event listener registered for Laravel's `RouteMatched` event. Triggers tenant resolution at
the `ResolutionHook::Routing` point.

**Registration:** Conditional, in `SproutServiceProvider::registerEventListeners()`:

```php
if ($this->sprout->supportsHook(ResolutionHook::Routing)) {
    $events->listen(RouteMatched::class, IdentifyTenantOnRouting::class);
}
```

**Behaviour:**

1. Parses the matched route's middleware stack to locate Sprout middleware
2. Extracts resolver and tenancy names from middleware parameters
3. Delegates to `ResolutionHelper::handleResolution()` with `throw: false`

**Middleware Detection:** The listener searches for either:

- `sprout.tenanted` (`SproutTenantContextMiddleware::ALIAS`)
- `sprout.tenanted.optional` (`SproutOptionalTenantContextMiddleware::ALIAS`)

If no Sprout middleware is found on the route, resolution is skipped.

### SproutTenantContextMiddleware

**Location:** `Sprout\Core\Http\Middleware\SproutTenantContextMiddleware`

**Alias:** `sprout.tenanted`

Middleware that handles both the `Middleware` hook resolution and tenant enforcement.

**Dual Function:**

1. **Resolution** — If `ResolutionHook::Middleware` is enabled, attempts tenant resolution
2. **Enforcement** — Throws `NoTenantFoundException` if no tenant was resolved by any hook

**Behaviour:**

```php
public function handle(Request $request, Closure $next, string ...$options): Response
{
    [$resolverName, $tenancyName] = ResolutionHelper::parseOptions($options);

    if ($this->sprout->supportsHook(ResolutionHook::Middleware)) {
        ResolutionHelper::handleResolution(
            $request,
            ResolutionHook::Middleware,
            $this->sprout,
            $resolverName,
            $tenancyName,
        );
    }

    if (! $this->sprout->hasCurrentTenancy() || ! $this->sprout->getCurrentTenancy()?->check()) {
        throw NoTenantFoundException::make(
            $resolverName ?? $defaultResolver,
            $tenancyName ?? $defaultTenancy
        );
    }

    return $next($request);
}
```

### SproutOptionalTenantContextMiddleware

**Location:** `Sprout\Core\Http\Middleware\SproutOptionalTenantContextMiddleware`

**Alias:** `sprout.tenanted.optional`

Middleware that handles the `Middleware` hook resolution without tenant enforcement.

**Behaviour:**

- Attempts resolution if `ResolutionHook::Middleware` is enabled
- Does not throw an exception if no tenant is found
- Passes `optional: true` to `ResolutionHelper::handleResolution()`

## Resolution Helper

**Location:** `Sprout\Core\Support\ResolutionHelper`

A static helper class that orchestrates the resolution process for all hooks.

### Method: parseOptions()

Parses middleware parameters into resolver and tenancy names.

```php
public static function parseOptions(array $options): array
```

| Parameter Count | Result                          |
|-----------------|---------------------------------|
| 2               | `[$resolverName, $tenancyName]` |
| 1               | `[$resolverName, null]`         |
| 0               | `[null, null]`                  |

### Method: handleResolution()

The central resolution orchestration method.

```php
public static function handleResolution(
    Request        $request,
    ResolutionHook $hook,
    Sprout         $sprout,
    ?string        $resolverName = null,
    ?string        $tenancyName = null,
    bool           $throw = true,
    bool           $optional = false
): bool
```

**Parameters:**

| Parameter       | Type             | Description                                       |
|-----------------|------------------|---------------------------------------------------|
| `$request`      | `Request`        | The current HTTP request                          |
| `$hook`         | `ResolutionHook` | The hook being processed                          |
| `$sprout`       | `Sprout`         | The Sprout instance                               |
| `$resolverName` | `?string`        | Resolver name, or `null` for default              |
| `$tenancyName`  | `?string`        | Tenancy name, or `null` for default               |
| `$throw`        | `bool`           | Whether to throw on failure (default: `true`)     |
| `$optional`     | `bool`           | Whether resolution is optional (default: `false`) |

**Return Value:** `true` if a tenant was successfully identified, `false` otherwise.

**Resolution Flow:**

1. **Set current hook** — `$sprout->setCurrentHook($hook)`
2. **Validate hook support** — Throws `MisconfigurationException` if hook is disabled
3. **Get resolver and tenancy** — Uses defaults if names are `null`
4. **Check skip conditions:**
    - If tenancy already has a tenant (`$tenancy->check()`), returns `false`
    - If resolver cannot resolve at this hook (`!$resolver->canResolve(...)`), returns `false`
5. **Set current tenancy** — `$sprout->setCurrentTenancy($tenancy)`
6. **Resolve identity:**
    - If resolver implements `IdentityResolverUsesParameters` AND route has the parameter:
      calls `$resolver->resolveFromRoute()`
    - Otherwise: calls `$resolver->resolveFromRequest()`
7. **Handle optional resolution** — If `$optional` and no identity, returns `false`
8. **Record resolution metadata** — `$tenancy->resolvedVia($resolver)->resolvedAt($hook)`
9. **Identify tenant** — Calls `$tenancy->identify($identity)`
10. **Handle failure** — If no tenant found and `$throw`, throws `NoTenantFoundException`

**Parameter Removal:** For parameter-based resolvers, the route parameter is removed after resolution to prevent it from
appearing in controller method signatures:

```php
$route->forgetParameter($resolver->getRouteParameterName($tenancy));
```

## Configuration

**Location:** `config/sprout/core.php`

### hooks

An array of `ResolutionHook` enum cases indicating which hooks are enabled.

```php
'hooks' => [
    \Sprout\Core\Support\ResolutionHook::Routing,
    \Sprout\Core\Support\ResolutionHook::Middleware,
],
```

The order of hooks in the configuration array does not affect execution order. Each hook fires at its predetermined
point in the Laravel request lifecycle.

**Default:** Both `Routing` and `Middleware` hooks are enabled.

## Hook State Tracking

The `Sprout` class tracks the current resolution hook during request processing.

### Methods

| Method             | Return Type       | Description                                     |
|--------------------|-------------------|-------------------------------------------------|
| `setCurrentHook()` | `static`          | Sets the current hook (called by handlers)      |
| `getCurrentHook()` | `?ResolutionHook` | Returns the current hook, or `null` if none     |
| `isCurrentHook()`  | `bool`            | Checks if the current hook matches the argument |
| `supportsHook()`   | `bool`            | Checks if a hook is enabled in configuration    |

### Usage

```php
// During request processing
$currentHook = $sprout->getCurrentHook();

// Check if at a specific hook
if ($sprout->isCurrentHook(ResolutionHook::Routing)) {
    // ...
}

// Check if a hook is enabled
if ($sprout->supportsHook(ResolutionHook::Middleware)) {
    // ...
}
```

## Middleware Parameters

Both middleware classes accept optional parameters to specify the resolver and tenancy.

### Format

```
sprout.tenanted
sprout.tenanted:{resolver}
sprout.tenanted:{resolver},{tenancy}
```

### Examples

```php
// Default resolver and tenancy
Route::middleware('sprout.tenanted')->group(function () {
    // ...
});

// Specific resolver, default tenancy
Route::middleware('sprout.tenanted:subdomain')->group(function () {
    // ...
});

// Specific resolver and tenancy
Route::middleware('sprout.tenanted:subdomain,organisations')->group(function () {
    // ...
});

// Optional tenant context
Route::middleware('sprout.tenanted.optional:header')->group(function () {
    // ...
});
```

## Resolver Compatibility

Resolvers declare which hooks they support via the `canResolve()` method on the `IdentityResolver` contract.

| Resolver  | Routing | Middleware | Notes                       |
|-----------|---------|------------|-----------------------------|
| Subdomain | ✓       | ✓          | Works at either hook        |
| Path      | ✓       | ✓          | Works at either hook        |
| Header    | ✓       | ✓          | Works at either hook        |
| Cookie    | ✓       | ✓          | Works at either hook        |
| Session   | ✗       | ✓          | Requires session middleware |

The session resolver explicitly checks for the `Middleware` hook:

```php
public function canResolve(Request $request, Tenancy $tenancy, ResolutionHook $hook): bool
{
    return $hook === ResolutionHook::Middleware;
}
```

## Middleware Priority

When using the `Middleware` hook with resolvers that depend on other middleware (such as the session resolver), you may
need to adjust middleware priority.

**Example:** Ensuring Sprout middleware runs after `StartSession`:

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToPriorityList(
            \Illuminate\Session\Middleware\StartSession::class,
            \Sprout\Core\Http\Middleware\SproutTenantContextMiddleware::class
        );
    });
```

> **Note:** Only adjust middleware priority when using resolvers that depend on middleware-provided data (e.g., session
> resolver). Changing priority unnecessarily can cause issues with service overrides and other Sprout features.

## Exceptions

### MisconfigurationException

**Location:** `Sprout\Core\Exceptions\MisconfigurationException`

Thrown when attempting to use a hook that is not enabled in configuration.

**Factory Method:**

```php
public static function unsupportedHook(ResolutionHook $hook): self
```

### NoTenantFoundException

**Location:** `Sprout\Core\Exceptions\NoTenantFoundException`

Thrown by `SproutTenantContextMiddleware` when no tenant was resolved by any hook and the route requires a tenant.

**Factory Method:**

```php
public static function make(string $resolver, string $tenancy): self
```

## Common Patterns

### Standard Web Application

Enable both hooks with a resolver that works at the `Routing` hook:

```php
// config/sprout/core.php
'hooks' => [
    ResolutionHook::Routing,
    ResolutionHook::Middleware,
],
```

The tenant is resolved early and available throughout the middleware stack.

### Session-Based Identification

For applications using the session resolver:

```php
// config/sprout/core.php
'hooks' => [
    ResolutionHook::Middleware,
],
```

Or keep both enabled — the session resolver will skip the `Routing` hook via its `canResolve()` method.

### API with Early Resolution

For APIs that only need early resolution (header, cookie, path, subdomain):

```php
// config/sprout/core.php
'hooks' => [
    ResolutionHook::Routing,
],
```

The `Middleware` hook is not needed since these resolvers do not require session data.

## Tenancy Resolution Tracking

After resolution, the tenancy tracks how and when the tenant was resolved:

```php
// Get the resolver that performed resolution
$resolver = $tenancy->resolver();

// Get the hook where resolution occurred
$hook = $tenancy->hook();

// Check if resolution has occurred
if ($tenancy->wasResolved()) {
    // ...
}
```

These methods are populated by `ResolutionHelper::handleResolution()`:

```php
$tenancy->resolvedVia($resolver)->resolvedAt($hook);
```

## Related Documents

- [Identity Resolvers](./identity-resolvers.md) — How tenant identity is extracted from requests
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events and listeners during tenant activation
- [Service Overrides](./service-overrides.md) — Making Laravel services tenant-aware
