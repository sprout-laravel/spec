# Identity Resolvers

Identity resolvers extract tenant identifiers from HTTP requests. They examine some part of the request — a subdomain,
a path segment, a header, a cookie, or session data — and return the identifier string that represents the tenant.

Resolvers are deliberately separated from [tenant providers](./tenant-providers.md). A resolver's only job is to extract an identifier string
from a request. It knows nothing about how tenants are stored or loaded. This separation means the same resolver works
regardless of whether tenants live in Eloquent, a database table, or an external API. And the same provider works
regardless of whether the identifier came from a subdomain, a header, or a cookie.

## Two Categories of Resolvers

Resolvers fall into two categories based on how they interact with routing.

### Routing Resolvers

**Subdomain** and **path** resolvers embed the tenant identifier in the URL structure itself.

```
https://acme.example.com/dashboard     (subdomain)
https://example.com/acme/dashboard     (path)
```

These resolvers:

- Require specific route configuration (domain constraints, path prefixes)
- Use Laravel route parameters — the identifier is extracted during routing
- Affect how routes are defined and how URLs are generated
- Make the tenant visible in the URL, which users can see and bookmark

When you define a tenanted route group, routing resolvers configure the routes with the appropriate constraints. The
subdomain resolver adds a domain constraint. The path resolver adds a path prefix. Both register a route parameter
that captures the tenant identifier.

Routing resolvers also implement a fallback. If the route parameter isn't available (error pages, routes outside the
tenanted group, edge cases), they fall back to parsing the request directly — extracting the subdomain from the host
or the segment from the path. This ensures resolution can still work even when Laravel's routing didn't capture the
parameter.

### Non-Routing Resolvers

**Header**, **cookie**, and **session** resolvers extract identifiers from request metadata rather than the URL.

```
X-Tenant: acme                         (header)
Cookie: tenant=acme                    (cookie)
Session: ['tenant' => 'acme']          (session)
```

These resolvers:

- Don't affect route definitions
- Don't embed the identifier in URLs
- Are simpler to configure
- Keep the tenant "invisible" to users (useful for APIs or seamless switching)

Non-routing resolvers examine the request after routing completes. They read the header value, decrypt the cookie, or
fetch from the session. The URL stays clean, which is sometimes what you want (especially for APIs where tenancy is
determined by an API key or token).

## When Resolution Happens

Resolvers declare which resolution hooks they support. Most resolvers work at both hooks (Routing and Middleware), but
some have constraints.

The **session resolver** only works at the Middleware hook. Sessions aren't available during routing — the session
middleware hasn't run yet. The resolver checks this and refuses to resolve at the wrong hook.

The **cookie resolver** works at both hooks but behaves differently. At the Routing hook, cookies haven't been
decrypted by Laravel's middleware yet, so the resolver manually decrypts. At the Middleware hook, cookies are already
decrypted.

Before attempting resolution, the system calls `canResolve()` to check if the resolver can run at the current hook.
This prevents wasted work and confusing errors.

See [Resolution Hooks](./resolution-hooks.md) for details on when each hook fires.

## Setup Actions

After a tenant is identified, resolvers perform setup actions. This happens via a listener on the [tenant identification
event](./tenancy-lifecycle.md).

**URL defaults.** Routing resolvers register the tenant identifier as a URL default. This means generated URLs
automatically include the tenant without you specifying it every time:

```php
// Without URL defaults, you'd need:
route('dashboard', ['tenant' => $tenant->identifier]);

// With URL defaults (set by resolver setup):
route('dashboard');  // tenant parameter filled automatically
```

**Cookie management.** The cookie resolver queues a cookie containing the tenant identifier. On subsequent requests,
this cookie identifies the tenant. When the tenant is cleared, the cookie is expired.

**Session storage.** The session resolver stores the tenant identifier in the session. Like cookies, this provides
persistence across requests.

**Settings repository.** Path and subdomain resolvers store values in Sprout's settings repository (URL path, URL
domain). Service overrides and other components read these settings.

Setup also runs when the tenant is cleared (with a null tenant). This allows resolvers to clean up — expire cookies,
forget session keys, clear URL defaults.

## Built-in Resolvers

### Subdomain

Extracts the identifier from the subdomain portion of the hostname.

```
acme.example.com → "acme"
```

Choose this when you want tenants to have their own subdomains. It's visible, memorable, and users can bookmark
tenant-specific URLs. Requires wildcard DNS or explicit subdomain configuration.

Configure with your parent domain:

```php
'subdomain' => [
    'driver' => 'subdomain',
    'domain' => 'example.com',
],
```

### Path

Extracts the identifier from a URL path segment.

```
/acme/dashboard → "acme"
```

Choose this when subdomains aren't feasible (shared hosting, SSL constraints) or when you want all tenants under a
single domain. The tenant is still visible in the URL.

By default, uses the first path segment. Configure which segment if needed:

```php
'path' => [
    'driver'  => 'path',
    'segment' => 1,  // first segment
],
```

### Header

Extracts the identifier from an HTTP request header.

```
X-Tenant-Identifier: acme → "acme"
```

Choose this for APIs where the tenant is specified programmatically. The client sets the header; users never see it.
Good for machine-to-machine communication.

The resolver also adds the tenant identifier to responses, so clients can confirm which tenant was used.

### Cookie

Extracts the identifier from an encrypted cookie.

```
Cookie: Tenants-Identifier=<encrypted:acme> → "acme"
```

Choose this for persistent tenant selection across requests without embedding in URLs. Users visit once with an
explicit tenant (via another resolver), then the cookie maintains it.

Useful for "remember my tenant" functionality or when combining with another resolver as a fallback.

### Session

Extracts the identifier from session storage.

```
Session['multitenancy.tenants'] = 'acme' → "acme"
```

Choose this when tenant selection happens through application logic (user picks from a dropdown) rather than URL
structure. The session maintains the selection.

Only works at the Middleware hook — sessions aren't available during routing.

## Service Override Conflicts

Two resolvers conflict with their corresponding [service overrides](./service-overrides.md):

**Cookie resolver + cookie override.** The cookie override changes how Laravel's cookie service works, modifying paths
and domains globally. The cookie resolver needs predictable cookie behaviour. They can't coexist.

**Session resolver + session override.** The session override creates tenant-specific session storage. But the session
resolver needs to read the session *before* knowing the tenant (to determine which tenant's session to use). They can't
coexist.

These aren't bugs — they're fundamental design constraints. The resolver needs the service to work one way; the
override changes it to work another way. Choose one approach per service.

When you enable a conflicting combination, Sprout throws `CompatibilityException` at resolution time with a clear
message about the conflict.

## Route Configuration

Routing resolvers need to configure routes. When you use `Route::tenanted()`, Sprout calls each resolver's
`configureRoute()` method.

The subdomain resolver adds a domain constraint:

```php
// Internally does something like:
$route->domain('{tenant}.example.com');
```

The path resolver adds a prefix:

```php
// Internally does something like:
$route->prefix('{tenant}');
```

Both also apply regex pattern constraints if configured, ensuring the tenant parameter matches expected formats.

Non-routing resolvers don't configure routes — they have nothing to add.

## URL Generation

Generating URLs to tenanted routes requires the tenant identifier. Resolvers handle this through their `route()`
method.

Routing resolvers inject the tenant identifier into the URL parameters:

```php
$resolver->route('dashboard', $tenancy, $tenant, [], true);
// Returns: https://acme.example.com/dashboard
```

Non-routing resolvers just delegate to Laravel's `route()` helper — there's nothing to inject.

During setup, routing resolvers register URL defaults, so for the *current* tenant you don't need to think about it:

```php
route('dashboard');  // Works automatically within tenant context
```

For generating URLs to *other* tenants, use Sprout's route helper:

```php
sprout()->route('dashboard', $otherTenant);
```

## Configuration

Resolvers are configured under `multitenancy.resolvers`:

```php
'resolvers' => [
    'subdomain' => [
        'driver' => 'subdomain',
        'domain' => env('TENANTED_DOMAIN'),
    ],
    'api' => [
        'driver' => 'header',
        'header' => 'X-Tenant-ID',
    ],
],
```

Each resolver has a name (used when specifying which resolver a route uses) and a driver (which resolver class to
instantiate). Additional options depend on the driver.

### Placeholders

Configuration values support placeholders that are replaced at runtime:

- `{tenancy}`, `{resolver}` — Lowercase versions
- `{Tenancy}`, `{Resolver}` — Capitalised versions (ucfirst)
- `{TENANCY}`, `{RESOLVER}` — Uppercase versions

This allows reusable configurations:

```php
'header' => '{Tenancy}-Identifier',  // Becomes "Tenants-Identifier"
```

## Extension

Custom resolvers extend the system for identification methods Sprout doesn't include.

### Simple Resolver

Extend `BaseIdentityResolver` and implement `resolveFromRequest()`:

```php
class ApiKeyResolver extends BaseIdentityResolver
{
    public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
    {
        $apiKey = $request->header('X-API-Key');

        if ($apiKey) {
            // Look up tenant identifier from API key
            return $this->lookupTenantByApiKey($apiKey);
        }

        return null;
    }
}
```

### Parameter-Based Resolver

If your resolver embeds the identifier in the URL (like subdomain or path), implement
`IdentityResolverUsesParameters`. The `FindsIdentityInRouteParameter` trait provides most of the implementation:

```php
class CustomParameterResolver extends BaseIdentityResolver implements IdentityResolverUsesParameters
{
    use FindsIdentityInRouteParameter;

    public function __construct(string $name, array $hooks = [])
    {
        parent::__construct($name, $hooks);
        $this->initialiseRouteParameter(null, '{tenancy}');
    }

    public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
    {
        // Fallback when parameter isn't in route
    }

    public function configureRoute(RouteRegistrar $route, Tenancy $tenancy): void
    {
        // Set up route constraints
    }
}
```

### Registration

Register custom drivers with the resolver manager:

```php
$sprout->resolvers()->register('api-key', function (array $config, string $name) {
    return new ApiKeyResolver($name, $config);
});
```

## Constraints and Gotchas

**Session resolver requires Middleware hook.** It simply won't work at the Routing hook — sessions don't exist yet.
The resolver enforces this and refuses to resolve.

**Cookie resolver decrypts manually at Routing hook.** Laravel's cookie middleware hasn't run yet, so cookies are
still encrypted. The resolver handles this, but it's worth knowing if you're debugging.

**Override conflicts are runtime errors.** Sprout doesn't prevent you from configuring conflicting resolver/override
combinations. The error happens when resolution is attempted. Check your configuration if you see
`CompatibilityException`.

**Resolvers don't validate tenants.** A resolver returns whatever identifier it finds. If that identifier doesn't
correspond to a real tenant, the [provider](./tenant-providers.md) returns null, and the [tenancy](./tenancy.md) ends up without a tenant. Validation is the
application's responsibility.

## Related Documents

- [Tenancy](./tenancy.md) — How resolvers are linked to tenancies
- [Tenant Providers](./tenant-providers.md) — What happens after resolution
- [Resolution Hooks](./resolution-hooks.md) — When resolution occurs
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events after resolution
