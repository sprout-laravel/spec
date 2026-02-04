# RFC-0002: Canopy — Domain-Based Tenant Identification

- **Status:** Draft
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-04
- **Updated:** 2026-02-04

## Summary

Canopy provides custom domain support for Sprout, allowing tenants to access the application via their own domains. It
includes a domain management system, DNS verification and validation workflows, driver-based SSL certificate
provisioning, and built-in subdomain fallback.

## Motivation

Many SaaS applications allow tenants to use custom domains:

- `acme.com` instead of `acme.app.example.com`
- `app.globex.net` instead of `globex.app.example.com`

This improves branding, builds trust with end-users, and is often a requirement for enterprise customers.

Custom domain support introduces several challenges:

1. **Routing** — Determining which tenant a request belongs to when it arrives on an arbitrary domain
2. **Verification** — Ensuring a tenant actually controls the domain they're claiming
3. **Validation** — Confirming DNS records are correctly configured
4. **SSL** — Provisioning and managing certificates for tenant domains
5. **Fallback** — Handling tenants without custom domains via subdomain identification

Sprout Core already handles subdomain and path-based identification. Canopy extends this to support fully custom domains
while integrating with Core's tenancy system.

### Goals

- Enable tenants to use their own domains
- Provide DNS verification to prove domain ownership
- Provide DNS validation to confirm correct configuration
- Support driver-based SSL certificate management
- Include built-in subdomain fallback for tenants without custom domains
- Integrate cleanly with Core's resolver system

### Non-Goals

- Canopy does not replace Core's subdomain resolver (though it can fall back to it)
- Canopy does not manage DNS records (tenants configure their own DNS)
- Canopy does not handle tenant-controlled subdomains under custom domains (e.g., `*.acme.com`) — this is part of nested
  tenancies (separate RFC)

## Configuration

Canopy's configuration lives in `config/sprout/canopy.php`. The structure follows Sprout's conventions and mirrors
patterns from Laravel's `database.connections` and `auth.guards`.

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | Default Domain Repository
    |--------------------------------------------------------------------------
    |
    | The default domain repository to use when resolving custom domains.
    |
    */

    'default' => env('CANOPY_REPOSITORY', 'default'),

    /*
    |--------------------------------------------------------------------------
    | Domain Repositories
    |--------------------------------------------------------------------------
    |
    | Domain repositories define how custom domains are stored, looked up,
    | verified, validated, and secured. You can define multiple repositories
    | for different use cases (e.g., different SSL providers for different
    | environments).
    |
    */

    'repositories' => [

        'default' => [
            // The driver for domain lookup
            'driver' => 'eloquent',
            'model' => App\Models\Domain::class,

            // Caching for domain lookups
            'cache' => [
                'enabled' => true,
                'ttl' => 3600,
                'store' => null,
            ],

            // Subdomain fallback when domain isn't found
            'subdomain_fallback' => [
                'enabled' => true,
                'domain' => env('APP_DOMAIN', 'example.com'),
                'pattern' => '[a-z0-9][a-z0-9-]*',
            ],

            // DNS record type tenants should use
            'dns' => [
                'type' => 'cname', // 'cname' or 'a'

                // For CNAME records - hosts that should be pointed to
                // Supports placeholders: {tenancy}, {Tenancy}, {TENANCY}, {resolver}, {Resolver}, {RESOLVER}
                'hosts' => [
                    'domains.{tenancy}.example.com',
                ],

                // For A records - IP addresses that should be pointed to
                'ips' => [
                    // '203.0.113.50',
                    // '203.0.113.51',
                ],
            ],

            // Domain ownership verification (TXT record)
            'verification' => [
                'enabled' => true,
                'prefix' => '_sprout-verify', // TXT record: _sprout-verify.customdomain.com
            ],

            // SSL certificate provisioning (HTTP-01 challenge)
            'ssl' => [
                'enabled' => true,
                'driver' => 'letsencrypt', // 'letsencrypt', 'certbot', 'manual'

                'letsencrypt' => [
                    'directory' => 'production', // or 'staging'
                    'email' => env('CANOPY_SSL_EMAIL'),
                    'storage' => storage_path('canopy/ssl'),
                    // How to serve HTTP-01 challenge responses
                    'challenge' => 'route', // 'route' or 'file'
                    'challenge_path' => public_path('.well-known/acme-challenge'),
                ],

                'certbot' => [
                    'binary' => '/usr/bin/certbot',
                    'webroot' => public_path(),
                ],
            ],
        ],

    ],

    /*
    |--------------------------------------------------------------------------
    | Unknown Domain Handling
    |--------------------------------------------------------------------------
    |
    | How to handle requests to domains that aren't registered and don't
    | match the subdomain fallback pattern.
    |
    */

    'fallback' => [
        'strategy' => 'abort', // 'abort', 'redirect', 'handler'
        'status' => 404,
        'redirect_url' => null,
        'handler' => null, // App\Http\Handlers\UnknownDomainHandler::class
    ],

];
```

## Detailed Design

### Domain Resolution Flow

Canopy introduces a two-step resolution process with subdomain fallback (configured per-repository):

```
Request (hostname)
       │
       ▼
┌─────────────────────────────────┐
│     DomainRepository lookup     │
│   Is hostname a custom domain?  │
└────────┬───────────────┬────────┘
         │               │
      Found          Not found
         │               │
         ▼               ▼
┌─────────────────┐  ┌────────────────────────┐
│ Return tenant   │  │  Subdomain fallback    │
│ key from Domain │  │  (if enabled for       │
└─────────────────┘  │   this repository)      │
                     └───────┬───────┬────────┘
                             │       │
                        Matched   No match
                             │       │
                             ▼       ▼
                     Return subdomain  Unknown domain
                     as identifier     handling
```

This indirection (Domain → tenant key → Tenant) is intentional:

- Domains are a separate concern from tenants
- A tenant can have multiple domains
- Domains can have their own attributes (primary, verified, validated, SSL status)
- The domain lookup can be optimised/cached independently
- **Tenants retain their identifier** — Since domains reference tenants by key (not replacing the identifier), tenants
  can still be identified via subdomain fallback using their original identifier. A tenant with identifier `acme` can
  have custom domain `acme.com` while still being accessible at `acme.example.com`.

### Domain Model

Domains are stored separately from tenants:

```php
class Domain extends Model
{
    protected $fillable = [
        'hostname',
        'tenant_key',
        'is_primary',
        'is_verified',
        'verified_at',
        'is_validated',
        'validated_at',
        'ssl_provisioned',
        'ssl_expires_at',
    ];
    
    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class, 'tenant_key', 'key');
    }
}
```

Key fields:

- **hostname** — The full domain (e.g., `acme.com`, `app.globex.net`)
- **tenant_key** — Reference to the tenant (uses tenant's key, not foreign key)
- **is_primary** — Whether this is the tenant's primary domain (used for URL generation)
- **is_verified** — Whether domain ownership has been verified via TXT record
- **is_validated** — Whether DNS records (A/CNAME) are correctly configured
- **ssl_provisioned** / **ssl_expires_at** — SSL certificate status

### Domain Repository

The `DomainRepository` interface handles domain lifecycle management:

```php
interface DomainRepository
{
    /**
     * Create a new domain record.
     */
    public function create(string $domainName, Tenancy $tenancy, Tenant $tenant): Domain;

    /**
     * Retrieve a domain record by its name.
     */
    public function retrieveByName(string $domainName): ?Domain;

    /**
     * Retrieve all domain records for a tenant.
     */
    public function retrieveForTenant(Tenant $tenant): Collection;
}
```

Unlike `TenantProvider` (which is read-only — tenants are managed by the user), `DomainRepository` handles full
lifecycle management since domains are managed by Canopy.

Canopy ships with an Eloquent implementation, but users can implement custom repositories (e.g., for external domain
management systems, caching layers, etc.).

### Resolver Integration

Canopy provides a `domain` identity resolver that implements `IdentityResolver` and `IdentityResolverUsesParameters`.
The resolver uses a route parameter that captures the entire hostname, allowing both custom domains and subdomains to
work with the same route definition.

```php
// config/sprout/resolvers.php (or within multitenancy.resolvers)
'resolvers' => [
    'domain' => [
        'driver' => 'domain',
        'repository' => 'default', // References canopy.repositories.default
    ],
],
```

The repository configuration (in `canopy.repositories.default`) controls:

- Domain lookup (driver, model, caching)
- Subdomain fallback (enabled, domain, pattern)
- DNS settings (type, hosts/ips)
- Verification (TXT record prefix)
- SSL provisioning (driver, challenge method)

Routes are defined with a domain parameter that captures the full hostname:

```php
Route::tenanted(function () {
    Route::get('/dashboard', DashboardController::class)->name('dashboard');
}, 'domain', 'tenants');

// This creates routes that match:
// - acme.com/dashboard (custom domain)
// - acme.example.com/dashboard (subdomain fallback)
```

The resolver tracks whether resolution succeeded via domain lookup or subdomain fallback, storing the actual resolution
method on the tenancy via `resolvedVia()`.

### URL Generation

URL generation uses Sprout's existing `route()` method, which delegates to the resolver that identified the tenant:

```php
// Using the Sprout facade
Sprout::route('dashboard', $tenant);

// Using the helper
sprout()->route('dashboard', $tenant);

// With explicit resolver/tenancy
sprout()->route('dashboard', $tenant, 'domain', 'tenants');
```

The method signature from `Sprout.php`:

```php
public function route(
    string $name,
    Tenant $tenant,
    ?string $resolver = null,
    ?string $tenancy = null,
    array $parameters = [],
    bool $absolute = true
): string
```

When no resolver is specified, it uses the resolver that identified the current tenant (if available), falling back to
the default. This means:

- If tenant was identified via custom domain → URLs use the custom domain
- If tenant was identified via subdomain fallback → URLs use the subdomain

For generating URLs for a different tenant (not the current one), the resolver will check:

1. Does the tenant have a primary custom domain? → Use it
2. No custom domain? → Use subdomain format

### DNS Verification vs Validation

Canopy distinguishes between two DNS-related checks:

**Verification** — Proving domain ownership via TXT record

When a tenant claims a domain, they must add a TXT record to prove ownership:

```
_sprout-verify.acme.com TXT "sprout-verify=abc123token"
```

This prevents:

- Someone pointing `competitor.com` at your servers and claiming it
- Tenants claiming domains they don't control
- DNS hijacking scenarios

**Validation** — Confirming DNS records are correctly configured

After verification, Canopy checks that the domain's DNS records point to the correct destination:

For **CNAME** configuration:

```
acme.com CNAME domains.tenants.example.com
```

For **A** record configuration:

```
acme.com A 203.0.113.50
acme.com A 203.0.113.51
```

Validation ensures traffic will actually reach the application before SSL provisioning begins.

### DNS Configuration

The `dns` section of each repository defines what DNS configuration tenants should use:

```php
'dns' => [
    'type' => 'cname',
    'hosts' => [
        'domains.{tenancy}.example.com',
    ],
],
```

The `hosts` array supports Sprout's placeholder system:

| Placeholder  | Output                  | Example   |
|--------------|-------------------------|-----------|
| `{tenancy}`  | Lowercase tenancy name  | `tenants` |
| `{Tenancy}`  | Ucfirst tenancy name    | `Tenants` |
| `{TENANCY}`  | Uppercase tenancy name  | `TENANTS` |
| `{resolver}` | Lowercase resolver name | `domain`  |
| `{Resolver}` | Ucfirst resolver name   | `Domain`  |
| `{RESOLVER}` | Uppercase resolver name | `DOMAIN`  |

For A records:

```php
'dns' => [
    'type' => 'a',
    'ips' => [
        '203.0.113.50',
        '203.0.113.51',
    ],
],
```

### Verification Flow

```php
// Tenant registers a domain
$domain = Domain::create([
    'hostname' => 'acme.com',
    'tenant_key' => $tenant->getTenantKey(),
]);

// Generate verification token
$token = $domain->generateVerificationToken();

// Display instructions to tenant:
// "Add a TXT record: _sprout-verify.acme.com with value: sprout-verify={$token}"

// Later, verify the domain
$domain->verify(); // Checks DNS for TXT record

// If successful:
// $domain->is_verified = true
// $domain->verified_at = now()
```

### Validation Flow

```php
// After verification, validate DNS configuration
$domain->validate(); // Checks A or CNAME records

// If successful:
// $domain->is_validated = true
// $domain->validated_at = now()

// Validation can be re-run periodically to detect DNS changes
```

### SSL Certificate Management

Canopy provides driver-based SSL certificate provisioning. Both the `letsencrypt` and `certbot` drivers use the HTTP-01
challenge method, which requires serving a response at `/.well-known/acme-challenge/{token}`. SSL provisioning only
proceeds after both verification and validation succeed.

**Let's Encrypt Driver** — Uses the ACME protocol directly:

```php
'ssl' => [
    'driver' => 'letsencrypt',
    'letsencrypt' => [
        'directory' => 'production',
        'email' => 'ssl@example.com',
        'storage' => storage_path('canopy/ssl'),
        'challenge' => 'route',
        'challenge_path' => public_path('.well-known/acme-challenge'),
    ],
],
```

The `challenge` option controls how HTTP-01 challenge responses are served:

- **`route`** — Canopy registers a route to serve challenge responses dynamically. Challenge tokens are stored
  temporarily and served via the application. This works in all environments but requires requests to reach the
  application.

- **`file`** — Canopy writes physical files to `challenge_path` (typically `public/.well-known/acme-challenge/`). This
  allows the web server to serve challenges directly without hitting the application, which can be faster and works even
  if the application is down. Requires write access to the public directory.

**Certbot Driver** — Shells out to certbot:

```php
'ssl' => [
    'driver' => 'certbot',
    'certbot' => [
        'binary' => '/usr/bin/certbot',
        'webroot' => public_path(),
    ],
],
```

Certbot uses its own HTTP-01 challenge handling via the `--webroot` plugin, writing challenge files to the specified
directory. The web server must be configured to serve files from `{webroot}/.well-known/acme-challenge/`.

**Manual Driver** — For externally managed certificates:

```php
'ssl' => [
    'driver' => 'manual',
],
```

Tenants or admins upload certificates manually. Canopy tracks expiry for alerting but doesn't provision automatically.

### HTTP-01 Challenge Handling

For the `letsencrypt` driver with `challenge => 'route'`, Canopy registers a route:

```php
// Registered by Canopy automatically when challenge => 'route'
Route::get('.well-known/acme-challenge/{token}', [AcmeChallengeController::class, 'show'])
    ->withoutMiddleware(['*']); // No auth/tenant middleware
```

This route is tenant-agnostic — it serves challenge responses for any domain being verified.

For `challenge => 'file'` or when using the `certbot` driver, ensure your web server is configured to serve static files
from the `.well-known/acme-challenge` directory:

```nginx
# Nginx example
location /.well-known/acme-challenge/ {
    root /var/www/html/public;
    try_files $uri =404;
}
```

### Unknown Domain Handling

When a request arrives for a domain that isn't registered and doesn't match the subdomain fallback:

```php
'fallback' => [
    'strategy' => 'abort',
    'status' => 404,
],

// Or redirect
'fallback' => [
    'strategy' => 'redirect',
    'redirect_url' => 'https://example.com/invalid-domain',
],

// Or custom handler
'fallback' => [
    'strategy' => 'handler',
    'handler' => App\Http\Handlers\UnknownDomainHandler::class,
],
```

A custom handler receives the request and hostname:

```php
class UnknownDomainHandler implements FallbackHandler
{
    public function handle(Request $request, string $hostname): Response
    {
        // Log attempt
        // Show "domain not configured" page
        // Check if domain is pending verification
        // etc.
    }
}
```

### Events

| Event                      | When                                |
|----------------------------|-------------------------------------|
| `DomainCreated`            | A new domain is registered          |
| `DomainVerified`           | Domain verification (TXT) succeeded |
| `DomainVerificationFailed` | Domain verification failed          |
| `DomainValidated`          | DNS validation (A/CNAME) succeeded  |
| `DomainValidationFailed`   | DNS validation failed               |
| `DomainDeleted`            | A domain is removed                 |
| `SslProvisioning`          | SSL provisioning is starting        |
| `SslProvisioned`           | SSL certificate obtained            |
| `SslRenewalDue`            | Certificate approaching expiry      |
| `SslRenewed`               | Certificate renewed                 |
| `SslProvisioningFailed`    | SSL provisioning failed             |

### Artisan Commands

| Command                  | Description                                |
|--------------------------|--------------------------------------------|
| `sprout:domain:list`     | List all domains (optionally for a tenant) |
| `sprout:domain:add`      | Add a domain to a tenant                   |
| `sprout:domain:remove`   | Remove a domain                            |
| `sprout:domain:verify`   | Verify domain ownership (TXT record)       |
| `sprout:domain:validate` | Validate DNS configuration (A/CNAME)       |
| `sprout:domain:primary`  | Set a domain as primary                    |
| `sprout:ssl:provision`   | Provision SSL for a domain                 |
| `sprout:ssl:renew`       | Renew expiring certificates                |
| `sprout:ssl:status`      | Show SSL status for domains                |

## Open Questions

### HTTP-01 Challenge Method Default

Should the default be `route` or `file` for the Let's Encrypt driver?

- **`route`** is simpler (no filesystem permissions needed) but requires the app to handle every challenge request
- **`file`** is faster and more resilient but requires write access to public directory

Likely `route` as default since it works in more environments out of the box.

### Certificate Storage in Distributed Environments

In load-balanced environments:

- Certificates need to be accessible from all servers
- ACME challenges need to be served from any server

Options:

- Store certificates in shared filesystem (NFS, EFS)
- Store in database (blob storage)
- Use external certificate manager (AWS ACM, Cloudflare)
- Document as infrastructure concern, out of scope

### Wildcard Certificate Support

Should Canopy support wildcard certificates for tenants wanting `*.acme.com`? This requires DNS-01 challenges which need
DNS API access. Likely out of scope for initial implementation.

### Verification Token Generation

How should verification tokens be generated and stored?

- Random string per domain
- HMAC based on domain + tenant + secret (stateless)
- Stored in domain record vs computed on demand

### Re-validation Scheduling

How often should DNS validation be re-checked?

- On every request (expensive)
- Scheduled job (e.g., daily)
- Only on SSL renewal
- Manual only

### Domain Transfer Between Tenants

What happens if a domain needs to move between tenants?

- Require re-verification
- Allow admin override
- Soft delete and recreate

## Alternatives Considered

### HTTP-based Verification

Could verify domains by checking for a file at `https://domain/.well-known/sprout-verification.txt`. Rejected because:

- Requires domain to already be pointing to us
- Chicken-and-egg problem with SSL
- DNS TXT is standard practice (Google, Let's Encrypt, etc.)

### External Domain Management Only

Could require users to manage domains entirely outside Sprout. Rejected because:

- Tight integration with resolution is valuable
- SSL provisioning needs domain awareness
- Verification workflows are common enough to include

### Separate Stacked Resolver for Fallback

Initially considered using RFC-0003's StackedResolver for domain + subdomain fallback. Rejected because:

- Added complexity for a common use case
- Both use same route parameter (full hostname)
- Built-in fallback is simpler and sufficient

## Implementation Plan

1. **Domain model and repository** — Core domain storage and lookup
2. **Resolver with subdomain fallback** — Identity resolution with built-in fallback
3. **DNS verification** — TXT record ownership verification
4. **DNS validation** — A/CNAME record configuration validation
5. **SSL provisioning** — Let's Encrypt and certbot drivers
6. **Artisan commands** — Management tooling
7. **Events** — Lifecycle events for domain management
