# Identity Resolvers

This document specifies how identity resolvers extract tenant identifiers from HTTP requests.

## Overview

Identity resolvers determine which tenant a request belongs to by examining request data and extracting an identifier.
This identifier is passed to a tenant provider, which loads the corresponding tenant entity.

### Key Concepts

- **Tenant**: An entity representing a distinct customer, organisation, or isolated context within the application.
  Implements `Sprout\Core\Contracts\Tenant`.
- **Tenancy**: Manages the current tenant state for a specific tenant type. A tenancy holds the current tenant, the
  provider used to load it, and the resolver used to identify it. Implements `Sprout\Core\Contracts\Tenancy`.
- **Tenant Provider**: Responsible for loading tenant entities from storage (database, cache, etc.) given an identifier
  or key. See [Tenant Providers](./tenant-providers.md).
- **Resolution Hook**: A point in the Laravel request lifecycle where resolution can occur (`Routing` or `Middleware`).
  See [Resolution Hooks](./resolution-hooks.md).

### Resolution Flow

```
Request
   │
   ▼
┌─────────────────────────┐
│   Identity Resolver     │
│   extracts identifier   │
│   (string or null)      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Tenant Provider      │
│    loads tenant entity  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Tenancy updated      │
│    setup() called       │
└───────────┬─────────────┘
            │
            ▼
    Tenant context active
```

Resolution is orchestrated by `ResolutionHelper::handleResolution()`, which:

1. Checks if the resolver can run at the current hook via `canResolve()`
2. Calls `resolveFromRoute()` (if parameter-based) or `resolveFromRequest()`
3. Passes the identifier to the tenancy's provider to load the tenant
4. Records which resolver and hook were used on the tenancy
5. Dispatches tenant lifecycle events, triggering `setup()` via `PerformIdentityResolverSetup`

## Contracts

### IdentityResolver

**Location:** `Sprout\Core\Contracts\IdentityResolver`

The base contract defines five methods:

| Method               | Signature                                                                                     | Purpose                                |
|----------------------|-----------------------------------------------------------------------------------------------|----------------------------------------|
| `resolveFromRequest` | `(Request $request, Tenancy $tenancy): ?string`                                               | Extract identifier from request        |
| `configureRoute`     | `(RouteRegistrar $route, Tenancy $tenancy): void`                                             | Configure route constraints/middleware |
| `setup`              | `(Tenancy $tenancy, ?Tenant $tenant): void`                                                   | Post-identification actions            |
| `canResolve`         | `(Request $request, Tenancy $tenancy, ResolutionHook $hook): bool`                            | Whether resolver can run at hook       |
| `route`              | `(string $name, Tenancy $tenancy, Tenant $tenant, array $parameters, bool $absolute): string` | Generate tenanted URLs                 |

#### Method Details

**`resolveFromRequest`**: The primary resolution method. Examines the request and returns a tenant identifier string, or
`null` if no identifier can be extracted. Returning `null` does not necessarily indicate an error — optional tenancy
routes may proceed without a tenant.

**`configureRoute`**: Called by `Route::tenanted()` when defining tenanted route groups. Allows the resolver to add
domain constraints, path prefixes, middleware, or route parameter patterns. Receives a `RouteRegistrar` for fluent
configuration.

**`setup`**: Called after tenant identification via the `PerformIdentityResolverSetup` listener. Receives `null` as the
tenant parameter when no tenant was identified (allowing cleanup). Used to configure URL defaults, queue cookies, update
session state, or set runtime settings.

**`canResolve`**: Determines if resolution should proceed. Called before `resolveFromRequest()`. Should return `false`
if the resolver cannot operate at the current hook (e.g., session resolver before session middleware) or if the tenancy
already has a tenant.

**`route`**: URL generation helper. Wraps Laravel's `route()` function, allowing resolvers to inject tenant-specific
parameters (e.g., subdomain, path prefix) into generated URLs.

### IdentityResolverUsesParameters

**Location:** `Sprout\Core\Contracts\IdentityResolverUsesParameters`

Extended contract for resolvers that extract identifiers from route parameters rather than parsing request data
directly.

| Method                  | Signature                                                     | Purpose                                 |
|-------------------------|---------------------------------------------------------------|-----------------------------------------|
| `getRouteParameterName` | `(Tenancy $tenancy): string`                                  | Get parameter name for tenancy          |
| `resolveFromRoute`      | `(Route $route, Tenancy $tenancy, Request $request): ?string` | Extract identifier from route parameter |

When a resolver implements this contract, `ResolutionHelper` checks for the route parameter first. If present, it calls
`resolveFromRoute()` and removes the parameter from the route (preventing it from appearing in controller method
signatures). If the parameter is absent, it falls back to `resolveFromRequest()`.

## Base Implementation

**Location:** `Sprout\Core\Support\BaseIdentityResolver`

Abstract class providing default implementations:

| Method           | Default Behaviour                                                     |
|------------------|-----------------------------------------------------------------------|
| `setup`          | No-op                                                                 |
| `configureRoute` | No-op                                                                 |
| `canResolve`     | Returns `true` if tenancy not resolved and hook is in supported hooks |
| `route`          | Delegates to Laravel's `route()` helper                               |

**Traits included:**

- `AwareOfSprout` — Provides `getSprout()` for accessing the Sprout instance
- `AwareOfApp` — Provides `getApp()` for accessing the Laravel application container

**Constructor:** `__construct(string $name, array $hooks = [])`

- `$name`: Resolver's registered name (used in configuration and placeholders)
- `$hooks`: Array of `ResolutionHook` cases this resolver supports. Defaults to `[ResolutionHook::Routing]`

## Configuration

Resolvers are configured in the Sprout configuration file under the `resolvers` key:

```php
// config/sprout.php
return [
    'resolvers' => [
        'resolver-name' => [
            'driver' => 'driver-name',  // Required: maps to a registered driver
            // ... driver-specific options
        ],
    ],
];
```

The `driver` key determines which resolver class to instantiate. Built-in drivers are `subdomain`, `path`, `header`,
`cookie`, and `session`. Custom drivers can be registered via `IdentityResolverManager::register()`.

### Placeholders

Configuration values that support placeholders use `PlaceholderHelper::replace()` for substitution at runtime:

| Placeholder  | Replaced With             |
|--------------|---------------------------|
| `{tenancy}`  | Tenancy name (lowercase)  |
| `{resolver}` | Resolver name (lowercase) |
| `{Tenancy}`  | Tenancy name (ucfirst)    |
| `{Resolver}` | Resolver name (ucfirst)   |
| `{TENANCY}`  | Tenancy name (UPPERCASE)  |
| `{RESOLVER}` | Resolver name (UPPERCASE) |

For route parameter names, `PlaceholderHelper::replaceForParameter()` additionally converts hyphens to underscores for
Laravel route compatibility.

### Settings Repository

Several resolvers store runtime values in `SettingsRepository` (`Sprout\Core\Support\SettingsRepository`) during
`setup()`. These settings are used by service overrides and URL generation:

| Setting                          | Set By                    | Purpose                        |
|----------------------------------|---------------------------|--------------------------------|
| `SettingsRepository::URL_PATH`   | PathIdentityResolver      | Path prefix for URL generation |
| `SettingsRepository::URL_DOMAIN` | SubdomainIdentityResolver | Domain for URL generation      |

Access via `$sprout->settings()`.

## Built-in Resolvers

### SubdomainIdentityResolver

**Location:** `Sprout\Core\Http\Resolvers\SubdomainIdentityResolver`

Extracts identifier from the subdomain portion of the request hostname.

**Implements:** `IdentityResolverUsesParameters` (via `FindsIdentityInRouteParameter` trait)

**Configuration:**

| Key         | Type   | Required | Default                | Placeholders | Description                          |
|-------------|--------|----------|------------------------|--------------|--------------------------------------|
| `domain`    | string | Yes      | —                      | No           | Parent domain (e.g., `example.com`)  |
| `pattern`   | string | No       | `null`                 | No           | Regex constraint for route parameter |
| `parameter` | string | No       | `{tenancy}_{resolver}` | Yes          | Route parameter name                 |
| `hooks`     | array  | No       | `[Routing]`            | No           | Supported resolution hooks           |

**Resolution:**

- `resolveFromRoute()`: Reads identifier from route parameter (trait implementation)
- `resolveFromRequest()`: Parses `$request->getHost()`, extracts substring before `.{domain}`

**Route configuration:** Calls `$route->domain()` with parameter placeholder and applies pattern constraint if
configured.

**Setup:** Sets `SettingsRepository::URL_DOMAIN` to the full tenant domain (e.g., `acme.example.com`). Also sets URL
default for the route parameter via `URL::defaults()`.

---

### PathIdentityResolver

**Location:** `Sprout\Core\Http\Resolvers\PathIdentityResolver`

Extracts identifier from a URL path segment.

**Implements:** `IdentityResolverUsesParameters` (via `FindsIdentityInRouteParameter` trait)

**Configuration:**

| Key         | Type   | Required | Default                | Placeholders | Description                             |
|-------------|--------|----------|------------------------|--------------|-----------------------------------------|
| `segment`   | int    | No       | `1`                    | No           | Path segment index (1-based, minimum 1) |
| `pattern`   | string | No       | `null`                 | No           | Regex constraint for route parameter    |
| `parameter` | string | No       | `{tenancy}_{resolver}` | Yes          | Route parameter name                    |
| `hooks`     | array  | No       | `[Routing]`            | No           | Supported resolution hooks              |

**Resolution:**

- `resolveFromRoute()`: Reads identifier from route parameter (trait implementation)
- `resolveFromRequest()`: Calls `$request->segment($segment)`

**Route configuration:** Calls `$route->prefix()` with parameter placeholder and applies pattern constraint if
configured.

**Setup:** Sets `SettingsRepository::URL_PATH` to the tenant identifier. Also sets URL default for the route parameter
via `URL::defaults()`.

---

### HeaderIdentityResolver

**Location:** `Sprout\Core\Http\Resolvers\HeaderIdentityResolver`

Extracts identifier from an HTTP request header.

**Configuration:**

| Key      | Type   | Required | Default                | Placeholders | Description                |
|----------|--------|----------|------------------------|--------------|----------------------------|
| `header` | string | No       | `{Tenancy}-Identifier` | Yes          | Header name                |
| `hooks`  | array  | No       | `[Routing]`            | No           | Supported resolution hooks |

**Resolution:** Calls `$request->header($headerName)` where `$headerName` has placeholders replaced.

**Route configuration:** Adds `AddTenantHeaderToResponse` middleware, which includes the tenant identifier in response
headers.

**Setup:** None.

---

### CookieIdentityResolver

**Location:** `Sprout\Core\Http\Resolvers\CookieIdentityResolver`

Extracts identifier from an encrypted cookie.

**Configuration:**

| Key       | Type   | Required | Default                | Placeholders | Description                |
|-----------|--------|----------|------------------------|--------------|----------------------------|
| `cookie`  | string | No       | `{Tenancy}-Identifier` | Yes          | Cookie name                |
| `options` | array  | No       | `[]`                   | No           | Cookie creation options    |
| `hooks`   | array  | No       | `[Routing]`            | No           | Supported resolution hooks |

**Cookie options:** `minutes`, `path`, `domain`, `secure`, `http_only`, `same_site`

**Resolution:**

1. Reads cookie via `$request->cookie($cookieName)`
2. If at `Routing` hook: manually decrypts using `Encrypter` (cookies not yet decrypted by middleware)
3. Validates using `CookieValuePrefix::validate()`

**Route configuration:** None.

**Setup:**

- If tenant exists: Queues cookie with identifier via `CookieJar::queue()`
- If tenant is null: Expires cookie via `CookieJar::expire()`

**Constraint:** Throws `CompatibilityException` if cookie service override is enabled. The override modifies cookie
behaviour globally, which interferes with this resolver's operation.

---

### SessionIdentityResolver

**Location:** `Sprout\Core\Http\Resolvers\SessionIdentityResolver`

Extracts identifier from session storage.

**Configuration:**

| Key       | Type   | Required | Default                  | Placeholders | Description |
|-----------|--------|----------|--------------------------|--------------|-------------|
| `session` | string | No       | `multitenancy.{tenancy}` | Yes          | Session key |

**Note:** Does not accept `hooks` configuration. Hardcoded to `ResolutionHook::Middleware` only, as sessions are
unavailable at earlier hooks.

**Resolution:** Calls `$request->session()->get($sessionKey)` where `$sessionKey` has placeholders replaced.

**Route configuration:** None.

**Setup:**

- If tenant exists: Stores identifier via `SessionManager::put()`
- If tenant is null: Removes key via `SessionManager::forget()`

**canResolve override:** Returns `true` only when all conditions are met:

- Tenancy not yet resolved
- Request has session (`$request->hasSession()`)
- Hook is `ResolutionHook::Middleware`

**Constraint:** Throws `CompatibilityException` if session service override is enabled. The override creates
tenant-specific session storage, but this resolver requires shared session access to determine tenant identity.

## Manager

**Location:** `Sprout\Core\Managers\IdentityResolverManager`

Factory responsible for creating and caching resolver instances. Extends `BaseFactory`.

**Configuration path:** `multitenancy.resolvers.{name}`

**Built-in drivers:**

| Driver      | Creator Method            | Class                       |
|-------------|---------------------------|-----------------------------|
| `subdomain` | `createSubdomainResolver` | `SubdomainIdentityResolver` |
| `path`      | `createPathResolver`      | `PathIdentityResolver`      |
| `header`    | `createHeaderResolver`    | `HeaderIdentityResolver`    |
| `cookie`    | `createCookieResolver`    | `CookieIdentityResolver`    |
| `session`   | `createSessionResolver`   | `SessionIdentityResolver`   |

**Accessing the manager:**

```php
// Via Sprout instance (dependency injection)
$resolverManager = $sprout->resolvers();
$resolver = $resolverManager->get('subdomain');
```

**Custom driver registration:**

```php
$sprout->resolvers()->register('custom', function (array $config, string $name) {
    return new CustomIdentityResolver($name, $config);
});
```

The closure receives the configuration array (from `multitenancy.resolvers.{name}`) and the resolver name.

## Exceptions

| Exception                   | Thrown When                                            | Location                  |
|-----------------------------|--------------------------------------------------------|---------------------------|
| `CompatibilityException`    | Cookie resolver used with cookie override enabled      | `CookieIdentityResolver`  |
| `CompatibilityException`    | Session resolver used with session override enabled    | `SessionIdentityResolver` |
| `MisconfigurationException` | Required config missing (e.g., `domain` for subdomain) | `IdentityResolverManager` |
| `MisconfigurationException` | Invalid config value (e.g., `segment` < 1)             | `IdentityResolverManager` |

## Extension

### Creating Custom Resolvers

Extend `BaseIdentityResolver` and implement `resolveFromRequest()`:

```php
class CustomIdentityResolver extends BaseIdentityResolver
{
    public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
    {
        // Extract and return identifier, or null if not found
    }
}
```

### FindsIdentityInRouteParameter Trait

**Location:** `Sprout\Core\Concerns\FindsIdentityInRouteParameter`

For parameter-based resolvers, this trait provides the complete implementation of `IdentityResolverUsesParameters`. It
handles route parameter extraction, pattern constraints, URL generation, and setup actions.

**Properties:**

| Property     | Type    | Default                | Description                        |
|--------------|---------|------------------------|------------------------------------|
| `$pattern`   | ?string | `null`                 | Regex constraint for parameter     |
| `$parameter` | string  | `{tenancy}_{resolver}` | Parameter name (with placeholders) |

**Methods provided:**

| Method                                                    | Purpose                                                  |
|-----------------------------------------------------------|----------------------------------------------------------|
| `initialiseRouteParameter($pattern, $parameter)`          | Set pattern and parameter values in constructor          |
| `setPattern($pattern)`                                    | Set the regex pattern constraint                         |
| `setParameter($parameter)`                                | Set the parameter name template                          |
| `getPattern()`                                            | Get the current pattern (or null)                        |
| `hasPattern()`                                            | Check if a pattern is configured                         |
| `getParameter()`                                          | Get the raw parameter template                           |
| `getRouteParameterName($tenancy)`                         | Get resolved parameter name (placeholders replaced)      |
| `getRouteParameter($tenancy)`                             | Get parameter wrapped in braces for route definitions    |
| `getParameterPatternMapping($tenancy)`                    | Get `[paramName => pattern]` array for route constraints |
| `applyParameterPatternMapping($registrar, $tenancy)`      | Apply pattern constraint to route registrar              |
| `resolveFromRoute($route, $tenancy, $request)`            | Extract identifier from route parameter                  |
| `setup($tenancy, $tenant)`                                | Set URL defaults for the parameter                       |
| `route($name, $tenancy, $tenant, $parameters, $absolute)` | Generate URL with tenant parameter                       |

**Contract implementation:**

The trait implements all methods required by `IdentityResolverUsesParameters`:

- `getRouteParameterName()` — Returns the parameter name with placeholders resolved
- `resolveFromRoute()` — Extracts the identifier from the route parameter, falling back to `resolveFromRequest()` if
  the parameter is not present

**Setup behaviour:**

The trait's `setup()` method registers the tenant identifier as a URL default:

```php
URL::defaults([
    $this->getRouteParameterName($tenancy) => $tenant?->getTenantIdentifier(),
]);
```

This ensures URL generation automatically includes the tenant parameter without explicit specification.

**Usage example:**

```php
class CustomParameterResolver extends BaseIdentityResolver implements IdentityResolverUsesParameters
{
    use FindsIdentityInRouteParameter;

    public function __construct(string $name, ?string $pattern = null, ?string $parameter = null, array $hooks = [])
    {
        parent::__construct($name, $hooks);
        $this->initialiseRouteParameter($pattern, $parameter);
    }

    public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
    {
        // Fallback resolution when parameter not in route
        // This is called by the trait's resolveFromRoute() if parameter is absent
    }

    public function configureRoute(RouteRegistrar $route, Tenancy $tenancy): void
    {
        // Use trait helpers to configure the route
        $this->applyParameterPatternMapping(
            $route->prefix($this->getRouteParameter($tenancy)),
            $tenancy
        );
    }

    public function setup(Tenancy $tenancy, ?Tenant $tenant): void
    {
        // Call trait setup for URL defaults
        $this->traitSetup($tenancy, $tenant);  // If aliased, or use parent if extending

        // Add resolver-specific setup
        $this->getSprout()->settings()->setUrlPath($this->getTenantRoutePrefix($tenancy));
    }
}
```

**Note:** If your resolver needs custom `setup()` behaviour, you must alias the trait method and call it explicitly, as
both `SubdomainIdentityResolver` and `PathIdentityResolver` do:

```php
use FindsIdentityInRouteParameter {
    FindsIdentityInRouteParameter::setup as parameterSetup;
}

public function setup(Tenancy $tenancy, ?Tenant $tenant): void
{
    parent::setup($tenancy, $tenant);
    $this->parameterSetup($tenancy, $tenant);
    // Additional setup...
}
```

## Compatibility Matrix

### Hook Support

| Resolver  | Routing | Middleware | Notes                             |
|-----------|---------|------------|-----------------------------------|
| Subdomain | ✓       | ✓          |                                   |
| Path      | ✓       | ✓          |                                   |
| Header    | ✓       | ✓          |                                   |
| Cookie    | ✓       | ✓          | Manual decryption at Routing hook |
| Session   | ✗       | ✓          | Requires session middleware       |

### Service Override Conflicts

| Resolver | Conflicting Override | Reason                                           |
|----------|----------------------|--------------------------------------------------|
| Cookie   | `cookie`             | Override modifies global cookie behaviour        |
| Session  | `session`            | Override creates tenant-specific session storage |

Both conflicts throw `CompatibilityException` at resolution time.

## Related Documents

- [Resolution Hooks](./resolution-hooks.md) — When and how resolution is triggered
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events and listeners during tenant activation
- [Service Overrides](./service-overrides.md) — How services become tenant-aware
- [Tenant Providers](./tenant-providers.md) — Loading tenants from storage
