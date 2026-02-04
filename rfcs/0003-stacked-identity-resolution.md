# RFC-0003: Stacked Identity Resolution

- **Status:** Under Review
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-04
- **Updated:** 2026-02-04

> **Note:** During the drafting of this RFC, significant constraints were identified that limit the practical use cases
> for stacked resolution. The primary motivating use case — domain resolution with subdomain fallback — is better served
> by building fallback directly into Canopy's domain resolver (see RFC-0002). This RFC remains open for community feedback
> on whether a generic stacking mechanism is valuable for other use cases.

## Summary

This RFC proposes a `StackedResolver` for Core that allows multiple identity resolvers to be chained in order of
precedence. When a request arrives, resolvers are tried in sequence until one successfully identifies a tenant. The
tenancy records the actual resolver that succeeded (not the stacked resolver), preserving existing behaviour for URL
generation and setup.

**Important constraint:** Stacking multiple resolvers that use route parameters (subdomain, path, domain) has
significant limitations around route configuration and URL generation. The stack works best when combining non-parameter
resolvers (header, cookie, session) with at most one parameter-based resolver.

## Motivation

Applications often need to support multiple identification methods simultaneously:

- **Header + Subdomain**: API clients send a header, browser users use subdomains — routes are configured for subdomain,
  header is a transparent fallback
- **Cookie + Subdomain**: Remember the tenant in a cookie for subsequent requests, subdomain is the source of truth
- **Domain + Subdomain**: Custom domains take precedence, fall back to subdomain for tenants without custom domains (
  requires careful route handling)

Currently, each tenancy is configured with a single resolver. Supporting multiple methods requires workarounds or custom
resolver implementations that duplicate the stacking logic.

Core already has a `StackedOverride` for layering service overrides. A `StackedResolver` would follow the same pattern
for identity resolution.

### Goals

- Allow multiple resolvers to be stacked with defined precedence
- Record the actual child resolver that succeeded on the tenancy (not the stack)
- Preserve existing URL generation behaviour via `IdentityResolver::route()`
- Work transparently for non-parameter resolvers (header, cookie, session)
- Provide clear guidance on parameter-based resolver combinations

### Non-Goals

- This RFC does not change the `IdentityResolver` contract
- This RFC does not change how URL generation works (already handled by `IdentityResolver::route()`)
- This RFC does not change the `Tenancy` contract
- This RFC does not solve the fundamental incompatibility between different route structures

## Fundamental Constraints

Before detailing the design, it's important to understand a fundamental limitation of stacking resolvers.

### Resolver Categories

Identity resolvers fall into two categories based on how they interact with routing:

**Non-routing resolvers** — These extract the identifier from request metadata without affecting route definitions:

- Header resolver — reads from HTTP header
- Cookie resolver — reads from cookie
- Session resolver — reads from session

**Routing resolvers** — These require specific route configuration and embed the identifier in the URL:

- Subdomain resolver — requires `->domain('{tenant}.example.com')`, identifier is in the hostname
- Path resolver — requires `->prefix('{tenant}')`, identifier is in the URL path
- Domain resolver (Canopy) — identifier is the entire hostname

### The Stacking Problem

Stacking only works cleanly in limited scenarios:

**Works well: Non-routing resolvers only**

```php
'resolvers' => ['header', 'cookie', 'session']
```

Routes are defined normally. Each resolver tries to find an identifier from its source. URL generation is unaffected
since none of these resolvers modify URLs.

**Works with caveats: One routing resolver + non-routing fallbacks**

```php
'resolvers' => ['subdomain', 'header']
```

Routes are defined with subdomain configuration. The subdomain resolver is "primary" for routing. Header acts as a
fallback for API clients that can't use subdomains. However:

- Requests via header might not match subdomain-configured routes
- URL generation uses subdomain format even for header-identified requests

**Does not work: Multiple routing resolvers**

```php
'resolvers' => ['subdomain', 'path']  // Problematic
```

These resolvers have fundamentally incompatible route configurations:

- Subdomain wants `->domain('{tenant}.example.com')`
- Path wants `->prefix('{tenant}')`

You cannot define a single route that works for both.

**Does not work: Path resolver in any stack**

The path resolver is uniquely problematic because the tenant identifier is embedded in the URL path. There's no way to:

- Generate a URL for a path-based tenant without the route being defined with that prefix
- Have routes match both `/dashboard` and `/{tenant}/dashboard`

With subdomain, you can at least generate a relative URL and prepend the domain. With path, the identifier IS part of
the path — there's no separation.

### Practical Implications

Given these constraints, `StackedResolver` is only suitable for:

1. **API flexibility** — Stack header + cookie + session for APIs that might identify via different mechanisms
2. **Subdomain with API fallback** — Subdomain as primary, header/cookie as fallback for clients that can't use
   subdomains
3. **Domain (Canopy) with subdomain fallback** — Custom domain as primary, fall back to subdomain for tenants without
   custom domains

The path resolver should generally NOT be used in a stack unless all other resolvers in the stack are non-routing
resolvers AND the path resolver is positioned as the primary (first) resolver.

### Alternative: Multiple Route Groups

For scenarios requiring truly different URL structures, the recommended approach is separate route groups rather than
stacking:

```php
// Subdomain-based routes
Route::sprout('subdomain', 'tenants', function () {
    Route::get('/dashboard', DashboardController::class);
});

// Path-based routes (different URLs, same controllers)
Route::sprout('path', 'tenants', function () {
    Route::get('/dashboard', DashboardController::class);
});
```

This duplicates route definitions but avoids the fundamental incompatibility.

## Detailed Design

### Configuration

A stacked resolver is configured by specifying an ordered list of resolvers:

```php
// config/sprout.php
'tenancies' => [
    'default' => [
        'resolver' => 'stack',
        // ...
    ],
],

'resolvers' => [
    'stack' => [
        'driver' => 'stack',
        'resolvers' => ['domain', 'subdomain', 'path'],
    ],
    
    'domain' => [
        'driver' => 'domain',
        // Canopy configuration...
    ],
    
    'subdomain' => [
        'driver' => 'subdomain',
        'domain' => env('APP_DOMAIN'),
    ],
    
    'path' => [
        'driver' => 'path',
        'segment' => 1,
    ],
],
```

Resolvers are tried in the order specified. The first resolver to return a non-null identifier wins.

### Resolution Flow

```
Request
   │
   ▼
┌─────────────────────────────────────────┐
│            StackedResolver              │
├─────────────────────────────────────────┤
│  1. Check 'domain' canResolve()         │
│     → true                              │
│     Call resolveFromRequest()           │
│     → null (no matching domain)         │
│                                         │
│  2. Check 'subdomain' canResolve()      │
│     → true                              │
│     Call resolveFromRequest()           │
│     → "acme" ✓                          │
│                                         │
│  3. Skip remaining resolvers            │
│                                         │
│  Return identifier: "acme"              │
│  Set tenancy->resolvedVia(subdomain)    │
└─────────────────────────────────────────┘
   │
   ▼
Tenancy uses identifier with TenantProvider
```

### StackedResolver Implementation

The `StackedResolver` implements `IdentityResolver` and delegates to child resolvers:

```php
class StackedResolver implements IdentityResolver
{
    /** @var list<IdentityResolver> */
    private array $resolvers;
    
    private ?IdentityResolver $matched = null;

    public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
    {
        foreach ($this->resolvers as $resolver) {
            if (! $resolver->canResolve($request, $tenancy, $hook)) {
                continue;
            }
            
            $identifier = $resolver->resolveFromRequest($request, $tenancy);
            
            if ($identifier !== null) {
                $this->matched = $resolver;
                return $identifier;
            }
        }
        
        return null;
    }
    
    public function getMatchedResolver(): ?IdentityResolver
    {
        return $this->matched;
    }
}
```

### Recording the Actual Resolver

The key behaviour: when the stacked resolver succeeds, the tenancy should record the *child* resolver that actually
matched, not the stacked resolver itself.

This requires coordination between the resolution process and the tenancy. Options:

**Option A: StackedResolver sets it directly**

The stacked resolver calls `$tenancy->resolvedVia()` with the matched child resolver before returning:

```php
public function resolveFromRequest(Request $request, Tenancy $tenancy): ?string
{
    foreach ($this->resolvers as $resolver) {
        // ...
        if ($identifier !== null) {
            $tenancy->resolvedVia($resolver);
            return $identifier;
        }
    }
    
    return null;
}
```

This works but has the resolver modifying tenancy state, which may be unexpected.

**Option B: Resolution orchestrator checks for stacking**

The code that calls `resolveFromRequest` and then `resolvedVia` checks if the resolver is a stack and unwraps:

```php
$identifier = $resolver->resolveFromRequest($request, $tenancy);

if ($identifier !== null) {
    $actualResolver = $resolver instanceof StackedResolver 
        ? $resolver->getMatchedResolver() 
        : $resolver;
    
    $tenancy->resolvedVia($actualResolver);
}
```

This keeps resolvers stateless but requires the orchestrator to know about stacking.

**Option C: StackedResolver is transparent**

The stacked resolver implements a marker interface, and `resolvedVia()` on the tenancy unwraps it:

```php
// In Tenancy implementation
public function resolvedVia(IdentityResolver $resolver): static
{
    if ($resolver instanceof StackedResolver) {
        $resolver = $resolver->getMatchedResolver();
    }
    
    $this->resolver = $resolver;
    return $this;
}
```

This keeps calling code unchanged but makes the tenancy aware of stacking.

### Delegating Contract Methods

The `StackedResolver` needs to implement all `IdentityResolver` methods. For methods that operate on the matched
resolver:

```php
public function setup(Tenancy $tenancy, ?Tenant $tenant): void
{
    // Delegate to matched resolver if available
    $this->matched?->setup($tenancy, $tenant);
}

public function route(string $name, Tenancy $tenancy, Tenant $tenant, array $parameters = [], bool $absolute = true): string
{
    if ($this->matched === null) {
        throw new RuntimeException('Cannot generate route: no resolver matched');
    }
    
    return $this->matched->route($name, $tenancy, $tenant, $parameters, $absolute);
}

public function configureRoute(RouteRegistrar $route, Tenancy $tenancy): void
{
    // Apply configuration from all resolvers that might match
    foreach ($this->resolvers as $resolver) {
        $resolver->configureRoute($route, $tenancy);
    }
}

public function canResolve(Request $request, Tenancy $tenancy, ResolutionHook $hook): bool
{
    // Can resolve if any child can resolve
    foreach ($this->resolvers as $resolver) {
        if ($resolver->canResolve($request, $tenancy, $hook)) {
            return true;
        }
    }
    
    return false;
}
```

### IdentityResolverUsesParameters

Some identity resolvers implement `IdentityResolverUsesParameters`, which adds `resolveFromRoute()` for route
parameter-based resolution. The subdomain resolver, path resolver, and Canopy's domain resolver all implement this
interface.

The `StackedResolver` implements `IdentityResolverUsesParameters` but proxies `resolveFromRoute()` calls to
`resolveFromRequest()`. This works because all resolvers that implement the parameters interface also provide a working
`resolveFromRequest()` implementation.

```php
class StackedResolver implements IdentityResolver, IdentityResolverUsesParameters
{
    public function resolveFromRoute(Route $route, Tenancy $tenancy, Request $request): ?string
    {
        // Proxy to request-based resolution, which all child resolvers support
        return $this->resolveFromRequest($request, $tenancy);
    }
    
    public function getRouteParameterName(Tenancy $tenancy): string
    {
        throw new BadMethodCallException(
            'StackedResolver does not have a single route parameter name. ' .
            'Child resolvers may have different parameter names.'
        );
    }
}
```

This means stacked resolution always uses the request-based path internally, even when triggered via route parameters.
The `getRouteParameterName()` method still throws since there's no single parameter name that applies — but this method
is only used during route configuration, not resolution.

### Route Configuration

When routes are configured with a stacked resolver, there's a fundamental constraint: **resolvers that use route
parameters have incompatible route structures**.

Consider a stack of `[domain, subdomain, path]`:

- **Domain resolver**: Routes match any domain, no parameters (e.g., `acme.com/dashboard`)
- **Subdomain resolver**: Routes need `{tenant}.example.com/dashboard`
- **Path resolver**: Routes need `example.com/{tenant}/dashboard`

These cannot be combined into a single route definition. A route registered with subdomain parameters won't match
path-based requests, and vice versa.

**Implication:** Stacked resolvers work best when:

1. **Non-parameter resolvers only**: Header, cookie, and session resolvers don't modify route structure. A stack of
   these works fine.

2. **One parameter resolver + non-parameter resolvers**: You can stack `[header, subdomain]` where subdomain is the "
   primary" and header is a fallback for API clients. Routes are configured for subdomain, header resolution just works
   because it doesn't need parameters.

3. **Separate route groups per resolver**: Register different route groups for each parameter-based resolver, each
   configured appropriately. The stack handles resolution, but route configuration is done per-resolver.

```php
// Option 3: Separate route groups
Route::tenanted('subdomain', function () {
    Route::get('/dashboard', DashboardController::class)->name('dashboard.subdomain');
});

Route::tenanted('path', function () {
    Route::get('/dashboard', DashboardController::class)->name('dashboard.path');
});

// Resolution uses the stack, routes are separate
```

**For `configureRoute()`:**

Given these constraints, the stacked resolver should NOT call `configureRoute()` on all children. Instead:

```php
public function configureRoute(RouteRegistrar $route, Tenancy $tenancy): void
{
    // Do nothing - route configuration must be handled per-resolver
    // when using parameter-based resolvers in a stack
}
```

Or, allow configuration of a "primary" resolver for route purposes:

```php
'resolvers' => [
    'stack' => [
        'driver' => 'stack',
        'resolvers' => ['domain', 'subdomain'],
        'routes' => 'subdomain', // Use subdomain for route configuration
    ],
],
```

```php
public function configureRoute(RouteRegistrar $route, Tenancy $tenancy): void
{
    // Delegate to the configured route resolver
    $this->routeResolver?->configureRoute($route, $tenancy);
}
```

### URL Generation Constraints

The `route()` method has the same constraint. The matched resolver's `route()` method expects routes to be configured in
a compatible way.

If a tenant was identified via subdomain but routes were configured for path-based resolution, URL generation will
produce incorrect URLs or fail.

**Recommendation:** When using stacked resolvers with parameter-based resolution, the resolver used for route
configuration should match the resolver most likely to be used. Alternatively, implement resolver-aware route naming:

```php
// Generate URL using the resolver that identified the tenant
$tenancy->resolver()->route('dashboard', $tenancy, $tenant);

// Or explicitly specify which resolver's URL format to use
sprout_route('dashboard', resolver: 'subdomain');
```

## Open Questions

### Is This Feature Worth the Complexity?

Given the significant constraints around parameter-based resolvers, it's worth asking whether a `StackedResolver`
provides enough value. The cleanest use cases are:

- `[header, <parameter-based>]` — API fallback
- `[cookie, <parameter-based>]` — Tenant persistence

These could potentially be handled by simpler mechanisms (e.g., a "fallback" option on parameter-based resolvers)
without a full stacking abstraction.

Counterargument: The stacking model is conceptually clean and matches `StackedOverride`. Even if practical use cases are
limited, consistency in the API has value.

### Which Option for Recording the Resolver?

Options A, B, and C each have trade-offs:

- **A** (resolver sets it): Simple but resolver modifies tenancy state
- **B** (orchestrator unwraps): Clean separation but orchestrator needs stacking awareness
- **C** (tenancy unwraps): Transparent to callers but tenancy needs stacking awareness

Recommendation: Option C feels most consistent with how the tenancy already manages this state.

### Is Stacking Parameter-Based Resolvers Supported?

Given the route configuration constraints, should stacking multiple parameter-based resolvers be:

- **Disallowed**: Throw at configuration time if stack contains multiple `IdentityResolverUsesParameters`
  implementations
- **Allowed with warnings**: Document the constraints, let users handle it
- **Allowed with primary designation**: Require a `routes` config to designate which resolver handles route
  configuration

The third option provides flexibility while making the constraints explicit.

### Separate Route Groups vs Single Routes

When using multiple parameter-based resolvers, should Sprout provide helpers for registering parallel route groups?

```php
// Hypothetical API
Route::tenantedStack(['subdomain', 'path'], function () {
    // Registers routes for both resolvers with appropriate names
    Route::get('/dashboard', DashboardController::class)->name('dashboard');
});

// Results in:
// - {tenant}.example.com/dashboard (name: subdomain.dashboard)
// - example.com/{tenant}/dashboard (name: path.dashboard)
```

This adds complexity but solves the URL generation problem cleanly.

### Precedence vs Specificity

Should resolvers be tried in configured order (precedence), or should there be a specificity system?

For example, if both domain and subdomain could resolve to the same tenant:

- **Precedence**: First in list wins (current proposal)
- **Specificity**: "More specific" matches win

Precedence is explicit and predictable. Specificity is implicit and might cause surprises. Recommendation: stick with
precedence.

### Generating Routes for Other Tenants

When `$tenancy->resolver()` returns the matched resolver, calling `route()` works for the current tenant. But what about
generating routes for a *different* tenant?

```php
// Current tenant was identified via subdomain
// Generate route for a tenant that has a custom domain
$tenancy->resolver()->route('dashboard', $tenancy, $otherTenant);
// Uses subdomain format, but $otherTenant might prefer domain format
```

This is arguably correct behaviour (generate URLs in the same format the user arrived with), but might not always be
desired.

## Drawbacks

### Route Configuration Complexity

The biggest drawback is that stacking parameter-based resolvers creates route configuration challenges. Users expecting
to freely combine subdomain + path + domain resolution will find that route registration and URL generation don't work
seamlessly. This is a fundamental limitation of how Laravel routing works, not something Sprout can fully abstract away.

### Mental Model Complexity

Users need to understand which resolver combinations work well together. The "stack anything" mental model is appealing
but misleading. Documentation must clearly explain:

- Non-parameter resolvers stack freely
- Parameter-based resolvers need careful handling
- Route configuration needs explicit consideration

### Limited Use Cases

Given the constraints, the practical use cases for stacking are narrower than they might appear:

- `[header, subdomain]` — API fallback for subdomain-based apps
- `[cookie, subdomain]` — Remember tenant across sessions
- `[domain, subdomain]` — Custom domains with subdomain fallback (but requires separate route groups)

More exotic combinations may not be worth the complexity.

## Alternatives Considered

### Composite Resolver per Application

Users write their own resolver combining strategies. Rejected because:

- Duplicates logic across applications
- No standard pattern for tracking which strategy matched
- Inconsistent approaches in the ecosystem

### Resolution Strategy on Tenant Model

Store preferred resolution method on the tenant. Rejected because:

- Conflates tenant data with routing concerns
- Doesn't reflect how the tenant was *actually* identified this request
- Complicates tenant model

## Implementation Plan

1. **StackedResolver class** — Core implementation with child resolver management
2. **Matched resolver tracking** — Mechanism to record which child succeeded
3. **Contract method delegation** — Proper delegation for setup, route, configureRoute
4. **Configuration** — Driver registration and config parsing
5. **Documentation** — Usage examples and guidance on resolver ordering
