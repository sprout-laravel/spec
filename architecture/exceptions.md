# Exceptions

Sprout uses a structured exception hierarchy to communicate different failure modes. Understanding when and why each
exception is thrown helps with debugging and building robust error handling.

## Design Philosophy

Sprout's exceptions follow several principles:

**Specific over generic.** Each exception type represents a distinct failure mode. Rather than throwing generic
exceptions with different messages, Sprout uses dedicated exception classes. This allows applications to catch specific
failures and handle them appropriately.

**Factory methods over constructors.** Most exceptions use static factory methods (`MisconfigurationException::missingConfig()`)
rather than public constructors. This keeps exception messages consistent and makes the codebase easier to search — you
can find everywhere a specific error is thrown by searching for the factory method name.

**Context in messages.** Exception messages include relevant context (which tenancy, which resolver, which config key)
to aid debugging. The factory methods ensure this context is always present and formatted consistently.

## Exception Hierarchy

All Sprout exceptions extend `SproutException`, which extends PHP's base `Exception`. This means you can catch all
Sprout exceptions with a single catch block when needed:

```php
try {
    // Tenanted operation
} catch (SproutException $e) {
    // Handle any Sprout failure
}
```

More commonly, you'll catch specific exception types to handle particular failure modes.

## Exception Categories

Sprout's exceptions fall into four categories based on when they occur and what they represent.

### Configuration Exceptions

These indicate problems with how Sprout is configured. They typically surface early — during boot or the first request
that exercises the misconfigured code path.

**MisconfigurationException** covers configuration problems:

- Missing required configuration values
- Invalid configuration (wrong types, missing required keys)
- References to undefined providers, resolvers, or tenancies
- Missing default values when defaults are required
- Resolvers used at unsupported hooks (e.g., session resolver at routing hook)

When you see this exception, check your `multitenancy.php` config file. The message tells you exactly which
configuration key is problematic.

### Compatibility Exceptions

**CompatibilityException** indicates that two components can't work together. Currently, this covers resolver/override
conflicts:

- Cookie resolver + cookie override (the override changes cookie behaviour in ways that break the resolver)
- Session resolver + session override (the resolver needs to read the session before knowing which tenant's session to use)

These aren't bugs — they're fundamental design constraints. The exception message explains which components conflict and
why. See [Identity Resolvers](./identity-resolvers.md#service-override-conflicts) for details on these specific
conflicts.

### Runtime Exceptions

These occur during normal operation when something expected isn't available.

**NoTenantFoundException** is thrown when a route requires a tenant but none was resolved. This happens when:

- A request hits a `Route::tenanted()` route without a valid tenant identifier
- The identifier was present but the [provider](./tenant-providers.md) couldn't find a matching tenant

This is the most common exception in production. It means someone accessed a tenanted URL without proper identification.
Applications typically catch this to show a "tenant not found" page or redirect to tenant selection.

**TenancyMissingException** is thrown when code expects a current tenancy but none is active. This typically indicates
code running outside a tenanted context that shouldn't be — perhaps a queued job or command that forgot to establish
tenant context first.

**TenantMissingException** is thrown when a [tenancy](./tenancy.md) exists but has no current tenant. Similar to above,
but the tenancy is active — it just doesn't have a tenant set. This can happen in "possibly tenanted" routes where the
tenant is optional.

### Eloquent Exceptions

These relate to [tenant-aware models](./eloquent-integration.md) and their relationships.

**TenantMismatchException** is thrown when attempting to associate a child model with a tenant different from its
current owner. This protects data integrity — a child model can't silently switch tenants.

**TenantRelationException** covers problems with the tenant relationship on child models:

- The model doesn't define a tenant relation
- The model defines multiple tenant relations (ambiguous ownership)

These exceptions indicate model configuration problems rather than runtime data issues.

### Service Override Exceptions

**ServiceOverrideException** covers problems with [service overrides](./service-overrides.md):

- Attempting to boot an override that doesn't support booting
- An override was set up but not enabled (internal consistency check)

These typically indicate bugs in custom override implementations rather than configuration problems.

## Handling Strategies

### Configuration and Compatibility Exceptions

These should surface during development, not production. If they reach production, something is misconfigured in your
deployment. Let them bubble up and get logged — they need developer attention.

### NoTenantFoundException

This is the exception you'll handle most often. Common strategies:

```php
// In exception handler
public function render($request, Throwable $e)
{
    if ($e instanceof NoTenantFoundException) {
        return response()->view('errors.tenant-not-found', [], 404);
    }

    return parent::render($request, $e);
}
```

Or redirect to tenant selection if your app supports it.

### Tenancy/Tenant Missing Exceptions

These indicate code running in the wrong context. In commands or jobs, establish tenant context before running tenanted
logic:

```php
$tenancy = $sprout->tenancies()->get('tenants');
$tenancy->setTenant($tenant);

// Now tenanted code can run
```

### Eloquent Exceptions

TenantMismatchException indicates a logic error — your code is trying to move data between tenants. Review the code
path that triggered it.

TenantRelationException indicates a model configuration issue. Check that tenant child models define exactly one
tenant relationship.

## Related Documents

- [Tenancy](./tenancy.md) — Tenant state management
- [Identity Resolvers](./identity-resolvers.md) — How tenants are identified
- [Service Overrides](./service-overrides.md) — Making services tenant-aware
- [Eloquent Integration](./eloquent-integration.md) — Tenant-aware models
