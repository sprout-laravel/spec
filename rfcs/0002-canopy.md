# RFC-0002: Canopy — Domain-Based Tenant Identification

- **Status:** Draft
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-04
- **Updated:** 2026-02-04

## Summary

Canopy provides custom domain support for Sprout, allowing tenants to access the application via their own domains. It
includes a domain management system, optional verification workflows, driver-based SSL certificate provisioning, and
fallback handling for unrecognised domains.

## Motivation

Many SaaS applications allow tenants to use custom domains:

- `acme.com` instead of `acme.app.example.com`
- `app.globex.net` instead of `globex.app.example.com`

This improves branding, builds trust with end-users, and is often a requirement for enterprise customers.

Custom domain support introduces several challenges:

1. **Routing** — Determining which tenant a request belongs to when it arrives on an arbitrary domain
2. **Verification** — Ensuring a tenant actually controls the domain they're claiming
3. **SSL** — Provisioning and managing certificates for tenant domains
4. **Fallback** — Handling requests to domains that aren't registered

Sprout Core already handles subdomain and path-based identification. Canopy extends this to support to fully custom
domains while integrating with Core's tenancy system.

### Goals

- Enable tenants to use their own domains
- Provide optional domain verification to prevent domain squatting
- Support driver-based SSL certificate management
- Handle fallback scenarios gracefully
- Integrate cleanly with Core's resolver system

### Non-Goals

- Canopy does not replace Core's subdomain resolver (that's already handled)
- Canopy does not manage DNS (tenants configure their own DNS)
- Canopy does not handle tenant-controlled subdomains under custom domains (e.g., `*.acme.com`) — this is part of nested
  tenancies (separate RFC)

## Detailed Design

### Domain Resolution Flow

Canopy introduces a two-step resolution process:

```
Request (acme.com)
       │
       ▼
┌─────────────────┐
│ DomainProvider  │ ──→ Find Domain by hostname
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Domain      │ ──→ Contains tenant key
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TenantProvider  │ ──→ Find Tenant by key
└────────┬────────┘
         │
         ▼
      Tenant
```

This indirection (Domain → tenant key → Tenant) is intentional:

- Domains are a separate concern from tenants
- A tenant can have multiple domains
- Domains can have their own attributes (primary, verified, SSL status)
- The domain lookup can be optimised/cached independently

### Domain Model

Domains are stored separately from tenants:

```php
class Domain extends Model
{
    protected $fillable = [
        'hostname',
        'tenant_key',
        'is_primary',
        'is_fallback',
        'is_verified',
        'verified_at',
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
- **tenant_key** — Reference to the tenant (not a foreign key, uses tenant's identifier)
- **is_primary** — Whether this is the tenant's primary domain (used for URL generation)
- **is_fallback** — Whether unverified requests should fall back to this domain
- **is_verified** — Whether domain ownership has been verified
- **ssl_provisioned** / **ssl_expires_at** — SSL certificate status

### Domain Provider

The `DomainProvider` interface abstracts domain lookup:

```php
interface DomainProvider
{
    public function findByHostname(string $hostname): ?Domain;
}
```

Canopy ships with an Eloquent implementation, but users can implement custom providers (e.g., for external domain
management systems, caching layers, etc.).

```php
// config/sprout.php
'canopy' => [
    'provider' => [
        'driver' => 'eloquent',
        'model' => App\Models\Domain::class,
        'cache' => [
            'enabled' => true,
            'ttl' => 3600,
            'store' => null, // Use default cache store
        ],
    ],
],
```

### Resolver Integration

Canopy provides a `domain` identity resolver that plugs into Core:

```php
// config/sprout.php
'tenancies' => [
    'default' => [
        'resolver' => 'domain',
        // ...
    ],
],

'resolvers' => [
    'domain' => [
        'driver' => 'domain',
        'provider' => 'eloquent',
    ],
],
```

The resolver:

1. Extracts hostname from the request
2. Queries the DomainProvider
3. Returns the tenant key from the Domain (if found)
4. Core's TenantProvider then loads the actual Tenant

### Fallback Handling

When a request arrives for an unknown domain, Canopy needs to handle it gracefully. Options are configurable:

```php
'canopy' => [
    'fallback' => [
        'strategy' => 'abort', // 'abort', 'redirect', 'handler'
        
        // For 'abort'
        'status' => 404,
        
        // For 'redirect'
        'url' => 'https://example.com',
        
        // For 'handler'
        'handler' => App\Http\Handlers\UnknownDomainHandler::class,
    ],
],
```

A custom handler receives the request and hostname:

```php
class UnknownDomainHandler implements FallbackHandler
{
    public function handle(Request $request, string $hostname): Response
    {
        // Check if this looks like a tenant trying to set up a domain
        // Show a "domain not configured" page
        // Log for monitoring
        // etc.
    }
}
```

### Domain Verification

Verification is optional but recommended. It prevents:

- Someone pointing `competitor.com` at your servers and claiming it
- Tenants claiming domains they don't control
- DNS hijacking scenarios

Canopy supports DNS record-based verification:

**DNS Verification** — Tenant adds a TXT record:

```
_sprout.acme.com TXT "sprout-verify=abc123token"
```

Configuration:

```php
'canopy' => [
    'verification' => [
        'enabled' => true,
        
        // DNS-specific
        'dns' => [
            'prefix' => '_sprout',
            'attribute' => 'sprout-verify',
        ],
    ],
],
```

Verification can be triggered:

```php
// Programmatically
$domain->verify();

// Via Artisan
php artisan sprout:domain:verify acme.com
```

Unverified domains can be configured to:

- Not resolve at all (strict)
- Resolve but with a flag accessible in the application
- Resolve with a grace period

### SSL Certificate Management

Canopy provides driver-based SSL certificate provisioning:

```php
'canopy' => [
    'ssl' => [
        'enabled' => true,
        'driver' => 'letsencrypt', // 'letsencrypt', 'certbot', 'manual', or custom
        
        'letsencrypt' => [
            'directory' => 'production', // or 'staging'
            'email' => 'ssl@example.com',
            'storage' => storage_path('ssl'),
        ],
        
        'certbot' => [
            'binary' => '/usr/bin/certbot',
            'webroot' => public_path(),
        ],
    ],
],
```

**Let's Encrypt Driver** — Uses the ACME protocol directly:

- Handles HTTP-01 challenges via `.well-known/acme-challenge` routes
- Stores certificates in configurable location
- Tracks expiry and handles renewal

**Certbot Driver** — Shells out to certbot:

- For environments where certbot is already installed/configured
- Sprout manages the webroot challenge files
- Certbot handles the actual certificate operations

**Manual Driver** — For externally managed certificates:

- Tenants or admins upload certificates
- Canopy tracks expiry for alerting
- No automatic provisioning

The SSL subsystem provides routes/files for challenges:

```php
// Option 1: Route-based (Canopy registers this)
Route::get('.well-known/acme-challenge/{token}', [AcmeChallengeController::class, 'show']);

// Option 2: File-based (Canopy writes to public directory)
// Files written to public/.well-known/acme-challenge/
```

### URL Generation

When generating URLs for a tenant, Canopy uses the primary domain:

```php
// Get URL for current tenant
sprout_url('/dashboard');
// → https://acme.com/dashboard

// Get URL for specific tenant
sprout_url('/dashboard', $tenant);
// → https://globex.net/dashboard

// Falls back to configured app URL if no domain
```

### Artisan Commands

| Command                 | Description                                |
|-------------------------|--------------------------------------------|
| `sprout:domain:list`    | List all domains (optionally for a tenant) |
| `sprout:domain:add`     | Add a domain to a tenant                   |
| `sprout:domain:remove`  | Remove a domain                            |
| `sprout:domain:verify`  | Verify domain ownership                    |
| `sprout:domain:primary` | Set a domain as primary                    |
| `sprout:ssl:provision`  | Provision SSL for a domain                 |
| `sprout:ssl:renew`      | Renew expiring certificates                |
| `sprout:ssl:status`     | Show SSL status for domains                |

### Events

| Event                      | When                           |
|----------------------------|--------------------------------|
| `DomainCreated`            | A new domain is registered     |
| `DomainVerified`           | Domain verification succeeded  |
| `DomainVerificationFailed` | Domain verification failed     |
| `DomainDeleted`            | A domain is removed            |
| `SslProvisioning`          | SSL provisioning is starting   |
| `SslProvisioned`           | SSL certificate obtained       |
| `SslRenewalDue`            | Certificate approaching expiry |
| `SslRenewed`               | Certificate renewed            |

## Open Questions

### Challenge Route vs File

For ACME HTTP-01 challenges, should Canopy:

- Register a route that serves challenge responses from storage
- Write actual files to the public directory

Routes are cleaner but require the application to handle every request. Files work with static file serving but require
write access to public directory.

### Certificate Storage

Where should certificates be stored?

- Filesystem (simple, but not shared in load-balanced setups)
- Database (shared, but certificates are large-ish)
- External service (e.g., AWS Certificate Manager, Cloudflare)

May need to support multiple storage backends.

### Wildcard Certificates

Should Canopy support wildcard certificates for tenants who want `*.acme.com`? This requires DNS-01 challenges which are
more complex (need DNS API access).

### Certificate Renewal Scheduling

How should automatic renewal be triggered?

- Scheduled command in the user's cron (`sprout:ssl:renew`)
- Background job dispatched when checking expiry
- Event-driven based on `SslRenewalDue` event

### Multi-Server Deployments

In load-balanced environments:

- ACME challenges need to be served from any server
- Certificates need to be distributed to all servers
- Renewal should only happen on one server

This may be out of scope (infrastructure concern), but needs consideration.

## Alternatives Considered

### Using Tenant Identifier as Domain

Could store domains directly on the Tenant model or use the identifier field. Rejected because:

- Tenants need multiple domains
- Domains have their own attributes (verified, SSL status, etc.)
- Separation of concerns — domains are a Canopy concept
- Tenants should always have a fallback method incase domains are not accessible

### External Domain Management Only

Could require users to manage domains entirely outside Sprout (e.g., in their own tables, external service). Rejected
because:

- Tight integration with resolution is valuable
- SSL provisioning needs domain awareness
- Verification workflows are common enough to include

### Single SSL Driver

Could only support Let's Encrypt or only certbot. Rejected because:

- Different environments have different constraints
- Enterprise users often have existing certificate management
- Flexibility is a Sprout principle

## Implementation Plan

1. **Domain model and provider** — Core domain storage and lookup
2. **Resolver integration** — Hook into Core's identity resolution
3. **Fallback handling** — Unknown domain responses
4. **Verification system** — DNS and HTTP verification drivers
5. **SSL provisioning** — Let's Encrypt and certbot drivers
6. **URL generation** — Helpers for domain-aware URLs
7. **Artisan commands** — Management tooling
