# RFC-0002: Canopy – Domain-Based Tenant Identification

- **Status:** Draft
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-04
- **Updated:** 2026-02-25

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

1. **Routing** – Determining which tenant a request belongs to when it arrives on an arbitrary domain
2. **Verification** – Ensuring a tenant actually controls the domain they're claiming
3. **Validation** – Confirming DNS records are correctly configured
4. **SSL** – Provisioning and managing certificates for tenant domains
5. **Fallback** – Handling tenants without custom domains via subdomain identification

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
- Canopy does not handle tenant-controlled subdomains under custom domains (e.g., `*.acme.com`) – this is part of nested
  tenancies (separate RFC)

## Configuration

Canopy's configuration is split across two locations: Canopy's own config file for infrastructure concerns (domain
storage, certificate provisioning, DNS lookup), and tenancy options within `multitenancy.php` for per-tenancy operational
settings.

### Canopy Config (`config/sprout/canopy.php`)

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | DNS Lookup Driver
    |--------------------------------------------------------------------------
    |
    | The driver used to perform DNS lookups for domain verification and
    | validation. The 'native' driver uses PHP's dns_get_record function.
    |
    */

    'lookup' => env('CANOPY_LOOKUP', 'native'),

    'lookups' => [
        'native' => [
            'driver' => 'native',
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Domain Repositories
    |--------------------------------------------------------------------------
    |
    | Domain repositories define how custom domains are stored and retrieved.
    | The repository is purely a storage concern – it does not handle
    | verification, validation, or SSL.
    |
    */

    'repositories' => [
        'default' => [
            'driver' => 'eloquent',
            'model'  => App\Models\Domain::class,
            'cache'  => [
                'enabled' => true,
                'ttl'     => 3600,
                'store'   => null,
            ],
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Certificate Managers
    |--------------------------------------------------------------------------
    |
    | Certificate managers handle SSL certificate provisioning, renewal,
    | and storage. Each driver manages the full certificate lifecycle.
    |
    */

    'certificates' => [
        'letsencrypt' => [
            'driver'         => 'letsencrypt',
            'directory'      => 'production', // or 'staging'
            'email'          => env('CANOPY_SSL_EMAIL'),
            'storage'        => storage_path('canopy/ssl'),
            'challenge'      => 'route', // 'route' or 'file'
            'challenge_path' => public_path('.well-known/acme-challenge'),
        ],

        'certbot' => [
            'driver'  => 'certbot',
            'binary'  => '/usr/bin/certbot',
            'webroot' => public_path(),
        ],

        'manual' => [
            'driver' => 'manual',
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Unknown Domain Handling
    |--------------------------------------------------------------------------
    |
    | How to handle requests to domains that aren't registered and don't
    | match any subdomain fallback pattern.
    |
    */

    'fallback' => [
        'strategy'     => 'abort', // 'abort', 'redirect', 'handler'
        'status'       => 404,
        'redirect_url' => null,
        'handler'      => null, // App\Http\Handlers\UnknownDomainHandler::class
    ],

];
```

### Resolver Config (in `config/sprout/multitenancy.php`)

The domain resolver is configured alongside other resolvers, with fallback and resolution strictness settings:

```php
'resolvers' => [
    'domain' => [
        'driver'     => 'domain',
        'repository' => 'default', // References canopy.repositories.default
        'fallback'   => [
            'enabled' => true,
            'domain'  => env('APP_DOMAIN', 'example.com'),
            'pattern' => '[a-z0-9][a-z0-9-]*',
        ],
        'resolve' => [
            'unverified'     => 'reject',  // reject | allow
            'unvalidated'    => 'allow',   // reject | allow
            'stale_checksum' => 'allow',   // reject | allow
        ],
    ],
],
```

### Tenancy Options (in `config/sprout/multitenancy.php`)

Per-tenancy operational settings live under the tenancy's `options` key:

```php
'tenancies' => [
    'tenants' => [
        'provider' => 'tenants',
        'options'  => [
            'canopy.repository'   => 'default',
            'canopy.certificates' => 'letsencrypt', // or null to disable SSL
            'canopy.domains'      => 'single',      // 'single' or 'multiple'

            'canopy.dns' => [
                'match'   => 'any', // 'any' or 'all'
                'records' => [
                    ['type' => 'CNAME', 'values' => ['domains.tenants.example.com']],
                ],
            ],

            'canopy.verification' => [
                'enabled' => true,
                'prefix'  => '_sprout-verify',
            ],
        ],
    ],
],
```

## Detailed Design

### Contracts

#### Domain

The `Domain` contract is a thin read surface representing a domain record. All state transitions are handled by the
`DomainRepository` – the domain itself is passive.

```php
interface Domain
{
    public function getHostname(): string;
    public function getTenantKey(): string;
    public function getTenancyName(): string;

    public function isPrimary(): bool;

    public function isVerified(): bool;
    public function getVerifiedAt(): ?DateTimeInterface;
    public function getVerificationToken(): ?string;
    public function getVerificationChecksum(): ?string;

    public function isValidated(): bool;
    public function getValidatedAt(): ?DateTimeInterface;
    public function getValidationChecksum(): ?string;

    public function isSslPending(): bool;
    public function isSslProvisioned(): bool;
    public function getSslExpiresAt(): ?DateTimeInterface;
}
```

#### Domain Repository

The `DomainRepository` handles domain storage and state transitions. Following the pattern established by Bud's
`ConfigStore`, the tenancy is provided per-call rather than being set on the instance, keeping the repository stateless.

```php
interface DomainRepository
{
    public function getName(): string;

    // Core CRUD
    public function create(
        Tenancy $tenancy,
        Tenant  $tenant,
        string  $hostname
    ): ?Domain;

    public function findByHostname(
        Tenancy $tenancy,
        string  $hostname
    ): ?Domain;

    public function findForTenant(
        Tenancy $tenancy,
        Tenant  $tenant
    ): Collection;

    public function delete(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    // Verification
    public function markVerified(
        Tenancy $tenancy,
        Domain  $domain,
        string  $checksum
    ): bool;

    public function markUnverified(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    // Validation
    public function markValidated(
        Tenancy $tenancy,
        Domain  $domain,
        string  $checksum
    ): bool;

    public function markInvalidated(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    // SSL
    public function markSslPending(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    public function markSslProvisioned(
        Tenancy           $tenancy,
        Domain            $domain,
        DateTimeInterface $expiresAt
    ): bool;

    public function markSslExpired(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    public function findWithExpiringSsl(
        Tenancy           $tenancy,
        DateTimeInterface $before
    ): Collection;

    public function findWithPendingSsl(
        Tenancy $tenancy
    ): Collection;

    // Bulk queries
    public function findVerified(Tenancy $tenancy): Collection;
    public function findValidated(Tenancy $tenancy): Collection;

    // Primary
    public function markPrimary(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;
}
```

The `create` method returns `?Domain` – returning `null` on failure (e.g., duplicate hostname) rather than throwing,
allowing consumers to handle expected failures gracefully.

Checksums are passed into `markVerified` and `markValidated` by the calling action, not generated by the repository.
This keeps the repository unaware of what the checksum represents.

#### DNS Lookup Driver

A pluggable driver for performing DNS lookups, used by both verification and validation. Configured globally in
`canopy.lookup` – a single driver is active at a time.

```php
enum DnsRecordType: string
{
    case A     = 'A';
    case AAAA  = 'AAAA';
    case CNAME = 'CNAME';
    case TXT   = 'TXT';
}
```

```php
interface DnsRecord
{
    public function getType(): string;
    public function getHostname(): string;
    public function getValue(): string;
    public function getTtl(): int;
}
```

```php
interface DnsLookupDriver
{
    public function getName(): string;

    /**
     * @return array<DnsRecord>
     */
    public function lookup(string $hostname, DnsRecordType $type): array;
}
```

Canopy ships with a `native` driver that wraps PHP's `dns_get_record`. The single-driver design keeps things simple
while still allowing it to be swapped (e.g., for a DoH driver, or a fake for testing).

#### Verification Token Generator

Responsible for generating the token tenants must add as a TXT record to prove domain ownership. Bound in the
container – implementations are swapped via container binding rather than configuration.

```php
interface VerificationTokenGenerator
{
    public function generate(Tenancy $tenancy, string $hostname): string;
}
```

Canopy ships a `RandomVerificationTokenGenerator` as the default.

#### Certificate Manager

Handles the full SSL certificate lifecycle – provisioning, renewal, revocation, and storage. Each driver manages its
own storage internally.

```php
interface CertificateManager
{
    public function getName(): string;

    public function provision(
        Tenancy $tenancy,
        Domain  $domain
    ): CertificateResult;

    public function renew(
        Tenancy $tenancy,
        Domain  $domain
    ): CertificateResult;

    public function revoke(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    public function hasCertificate(
        Tenancy $tenancy,
        Domain  $domain
    ): bool;

    public function getCertificate(
        Tenancy $tenancy,
        Domain  $domain
    ): ?Certificate;
}
```

For drivers that require polling (e.g., Let's Encrypt ACME), a child interface adds the ability to check progress:

```php
interface PollableCertificateManager extends CertificateManager
{
    public function check(
        Tenancy $tenancy,
        Domain  $domain
    ): CertificateResult;

    public function getPollInterval(): int; // seconds
}
```

The base `CertificateManager` is sufficient for synchronous drivers like Certbot. The `PollableCertificateManager`
adds `check` for drivers where provisioning is asynchronous, and `getPollInterval` to indicate how frequently the
consumer should poll. The poll interval is a characteristic of the driver, not of individual results.

#### Certificate Result

Represents the outcome of a provisioning, renewal, or check operation:

```php
interface CertificateResult
{
    public function isSuccessful(): bool;
    public function isPending(): bool;
    public function isFailed(): bool;

    public function getCertificate(): ?Certificate;
    public function getFailureReason(): ?string;
}
```

#### Certificate

Metadata and file paths for an issued certificate:

```php
interface Certificate
{
    public function getHostname(): string;
    public function getExpiresAt(): DateTimeInterface;
    public function getIssuedAt(): ?DateTimeInterface;
    public function getIssuer(): ?string;

    public function getCertificatePath(): ?string;
    public function getPrivateKeyPath(): ?string;
    public function getChainPath(): ?string;
}
```

Paths are nullable. For drivers where Canopy provisions the certificate, files will exist and paths will be populated.
For managed services or edge cases, paths may be null – consumers requiring file paths should handle this and throw
if a path is unexpectedly absent.

### Database Schema

```
domains
├── id
├── tenancy
├── hostname
├── tenant_key
├── is_primary
├── is_verified
├── verified_at
├── verification_token
├── verification_checksum
├── is_validated
├── validated_at
├── validation_checksum
├── is_ssl_pending
├── is_ssl_provisioned
├── ssl_expires_at
└── timestamps
```

The `tenancy` column scopes all queries. Two tenancies can share the same repository and table, differentiated by this
column. The repository name is not stored – it is an infrastructure detail, not a meaningful scope.

### Domain Resolution Flow

Canopy introduces a domain resolver implementing `IdentityResolver`. The resolver performs a two-step resolution
process with subdomain fallback:

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
│ Check domain    │  │  Subdomain fallback    │
│ state & resolve │  │  (if enabled on        │
│ options         │  │   this resolver)       │
└─────────────────┘  └───────┬───────┬────────┘
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
- Domains have their own attributes (primary, verified, validated, SSL status)
- The domain lookup can be optimised/cached independently
- **Tenants retain their identifier** – Since domains reference tenants by key (not replacing the identifier), tenants
  can still be identified via subdomain fallback using their original identifier

#### Resolution State Checks

When a domain is found, the resolver checks its state against the `resolve` options configured on the resolver:

```php
if (!$domain->isVerified()) {
    $this->fireEvent(new UnverifiedDomainResolution($tenancy, $domain));

    if ($this->shouldReject($tenancy, 'unverified')) {
        return null;
    }
}

if (!$domain->isValidated()) {
    $this->fireEvent(new UnvalidatedDomainResolution($tenancy, $domain));

    if ($this->shouldReject($tenancy, 'unvalidated')) {
        return null;
    }
}

if ($this->isChecksumStale($tenancy, $domain)) {
    $this->fireEvent(new StaleDomainResolution($tenancy, $domain));

    if ($this->shouldReject($tenancy, 'stale_checksum')) {
        return null;
    }
}
```

Events fire regardless of the allow/reject setting, so implementors can always observe domain state issues.

Default settings:

- **unverified: reject** – Security concern; domain ownership not proven
- **unvalidated: allow** – If traffic arrived, DNS is working
- **stale_checksum: allow** – Config changed but traffic is still arriving

#### Checksum Staleness

The domain record stores a checksum of the DNS configuration (or verification prefix) that was active at the time
of verification/validation. To detect staleness, the current config is hashed and compared:

```php
$dnsConfig = $tenancy->option('canopy.dns');
$currentChecksum = hash('xxh128', json_encode($dnsConfig));
$isStale = $domain->getValidationChecksum() !== $currentChecksum;
```

This allows passive detection of configuration drift without requiring DNS lookups.

#### Resolution Tracking

The resolver maintains a map of tenant keys to hostnames used during resolution:

```php
private array $resolvedVia = [];
```

When a tenant is resolved via custom domain, the hostname is stored. When resolved via subdomain fallback, the full
subdomain hostname is stored. This map is used during URL generation and when setting default route parameters.

### URL Generation

URL generation uses the resolver's stored resolution map when available, falling back to a repository lookup for tenants
that weren't resolved in the current request:

```php
public function route(
    string  $name,
    Tenancy $tenancy,
    Tenant  $tenant,
    array   $parameters = [],
    bool    $absolute = true
): string {
    $parameter = $this->getRouteParameterName($tenancy);

    if (!isset($parameters[$parameter])) {
        $key = $tenant->getTenantKey();

        if (isset($this->resolvedVia[$key])) {
            $parameters[$parameter] = $this->resolvedVia[$key];
        } else {
            $domain = $this->findUsableDomain($tenancy, $tenant);

            $parameters[$parameter] = $domain
                ? $domain->getHostname()
                : $tenant->getTenantIdentifier() . '.' . $this->fallbackDomain;
        }
    }

    return route($name, $parameters, $absolute);
}
```

The `findUsableDomain` method applies the same `resolve` options as resolution – if a domain wouldn't pass resolution
checks, it won't be used for URL generation either:

```php
private function findUsableDomain(Tenancy $tenancy, Tenant $tenant): ?Domain
{
    $domain = $this->repository
        ->findForTenant($tenancy, $tenant)
        ->first(fn (Domain $d) => $d->isPrimary());

    if ($domain === null) {
        return null;
    }

    if (!$domain->isVerified() && $this->shouldReject($tenancy, 'unverified')) {
        return null;
    }

    if (!$domain->isValidated() && $this->shouldReject($tenancy, 'unvalidated')) {
        return null;
    }

    if ($this->isChecksumStale($tenancy, $domain) && $this->shouldReject($tenancy, 'stale_checksum')) {
        return null;
    }

    return $domain;
}
```

### DNS Verification vs Validation

Canopy distinguishes between two DNS-related checks:

**Verification** – Proving domain ownership via TXT record.

When a tenant claims a domain, they must add a TXT record to prove ownership:

```
_sprout-verify.acme.com TXT "sprout-verify=abc123token"
```

This prevents:

- Someone pointing `competitor.com` at your servers and claiming it
- Tenants claiming domains they don't control
- DNS hijacking scenarios

Verification can be disabled per-tenancy via the `canopy.verification` option.

**Validation** – Confirming DNS records are correctly configured.

After verification, Canopy checks that the domain's DNS records match the tenancy's DNS configuration. The `dns`
tenancy option defines the expected records:

```php
// CNAME or A – only one needs to match
'dns' => [
    'match'   => 'any',
    'records' => [
        ['type' => 'A',     'values' => ['203.0.113.50']],
        ['type' => 'CNAME', 'values' => ['domains.tenants.example.com']],
    ],
],

// A and AAAA – both must be present
'dns' => [
    'match'   => 'all',
    'records' => [
        ['type' => 'A',    'values' => ['203.0.113.50']],
        ['type' => 'AAAA', 'values' => ['2001:db8::1']],
    ],
],
```

The `match` setting controls whether all records must be present (`all`) or at least one (`any`).

Validation ensures traffic will actually reach the application before SSL provisioning begins. On successful
verification or validation, a checksum of the relevant config is stored on the domain record to detect future
configuration drift.

### SSL Certificate Management

Canopy provides driver-based SSL certificate provisioning. SSL is enabled or disabled per-tenancy via the
`canopy.certificates` option – setting it to `null` disables SSL management entirely for that tenancy.

**Let's Encrypt Driver** – Uses the ACME protocol directly. Implements `PollableCertificateManager` as provisioning
is asynchronous:

```php
'letsencrypt' => [
    'driver'         => 'letsencrypt',
    'directory'      => 'production',
    'email'          => env('CANOPY_SSL_EMAIL'),
    'storage'        => storage_path('canopy/ssl'),
    'challenge'      => 'route',
    'challenge_path' => public_path('.well-known/acme-challenge'),
],
```

The `challenge` option controls how HTTP-01 challenge responses are served:

- **`route`** – Canopy registers a route to serve challenge responses dynamically. Challenge tokens are stored
  temporarily and served via the application. This works in all environments but requires requests to reach the
  application.
- **`file`** – Canopy writes physical files to `challenge_path` (typically `public/.well-known/acme-challenge/`). This
  allows the web server to serve challenges directly without hitting the application.

**Certbot Driver** – Shells out to certbot. Implements base `CertificateManager` as provisioning is synchronous:

```php
'certbot' => [
    'driver'  => 'certbot',
    'binary'  => '/usr/bin/certbot',
    'webroot' => public_path(),
],
```

**Manual Driver** – For externally managed certificates. Implements base `CertificateManager`, returns pending
from `provision` and waits for certificates to be provided through an external mechanism:

```php
'manual' => [
    'driver' => 'manual',
],
```

### HTTP-01 Challenge Handling

For the `letsencrypt` driver with `challenge => 'route'`, Canopy registers a route:

```php
Route::get('.well-known/acme-challenge/{token}', [AcmeChallengeController::class, 'show'])
    ->withoutMiddleware(['*']);
```

This route is tenant-agnostic – it serves challenge responses for any domain being verified.

For `challenge => 'file'` or when using the `certbot` driver, ensure your web server is configured to serve static files
from the `.well-known/acme-challenge` directory:

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/html/public;
    try_files $uri =404;
}
```

### Unknown Domain Handling

When a request arrives for a domain that isn't registered and doesn't match the subdomain fallback:

```php
// Abort with status code
'fallback' => [
    'strategy' => 'abort',
    'status'   => 404,
],

// Redirect
'fallback' => [
    'strategy'     => 'redirect',
    'redirect_url' => 'https://example.com/invalid-domain',
],

// Custom handler
'fallback' => [
    'strategy' => 'handler',
    'handler'  => App\Http\Handlers\UnknownDomainHandler::class,
],
```

A custom handler receives the request and hostname:

```php
class UnknownDomainHandler implements FallbackHandler
{
    public function handle(Request $request, string $hostname): Response
    {
        // Log attempt, show "domain not configured" page,
        // check if domain is pending verification, etc.
    }
}
```

### Actions

Actions are the primary API for interacting with Canopy. They orchestrate the contracts (repository, DNS lookup,
certificate manager) and fire events. Canopy provides action classes, queued job wrappers, and artisan commands for
each operation.

Each action returns an `ActionResult` (or a subclass with additional context):

```php
enum ActionStatus
{
    case Success;
    case Pending;
    case Failed;
}

class ActionResult
{
    public function __construct(
        public readonly ActionStatus $status,
        public readonly ?string $reason = null,
    ) {}

    public function isSuccessful(): bool
    {
        return $this->status === ActionStatus::Success;
    }

    public function isPending(): bool
    {
        return $this->status === ActionStatus::Pending;
    }

    public function isFailed(): bool
    {
        return $this->status === ActionStatus::Failed;
    }
}
```

Actions that return created objects use subclasses:

```php
class AddDomainResult extends ActionResult
{
    public function __construct(
        ActionStatus $status,
        public readonly ?Domain $domain = null,
        ?string $reason = null,
    ) {
        parent::__construct($status, $reason);
    }
}

class ProvisionCertificateResult extends ActionResult
{
    public function __construct(
        ActionStatus $status,
        public readonly ?Certificate $certificate = null,
        ?string $reason = null,
    ) {
        parent::__construct($status, $reason);
    }
}
```

#### Domain Lifecycle Actions

**`AddDomain`** – Creates a domain via the repository. Respects the `canopy.domains` tenancy option: in `single` mode,
replaces the existing domain; in `multiple` mode, adds alongside existing domains. Generates a verification token
(via `VerificationTokenGenerator`) if verification is enabled. Returns `AddDomainResult`.

**`RemoveDomain`** – Deletes a domain via the repository. Revokes the SSL certificate if one is provisioned.

**`SetPrimaryDomain`** – Marks a domain as primary via the repository. In `single` domain mode, the one domain is
always primary.

#### Verification Actions

**`VerifyDomain`** – Uses the DNS lookup driver to check the TXT record against the stored token and verification
prefix. Computes a checksum of the verification config and stores it on the domain. Fires success/failure events.

**`ReverifyDomains`** – Bulk re-check of all verified domains. Accepts an `async` flag: when true, dispatches a
`VerifyDomainJob` per domain; when false, runs each check inline.

#### Validation Actions

**`ValidateDomain`** – Uses the DNS lookup driver to check records against the tenancy's DNS configuration. Computes a
checksum and stores it on the domain. Fires success/failure events.

**`RevalidateDomains`** – Bulk re-check of all validated domains. Accepts an `async` flag.

**`InvalidateDomains`** – Bulk reset of validation status. Used after infrastructure changes. If the implementation
uses a single query, fires a bulk `DomainsInvalidated` event; if it iterates individually, fires `DomainInvalidated`
per domain.

#### SSL Actions

**`ProvisionCertificate`** – Checks SSL is enabled for the tenancy, checks domain is verified/validated, then calls
the certificate manager's `provision` method. Marks the domain as SSL pending or provisioned depending on the result.
Returns `ProvisionCertificateResult`.

**`CheckCertificate`** – For `PollableCertificateManager` drivers. Calls `check` and updates the domain accordingly.

**`RenewCertificate`** – Calls the certificate manager's `renew` method. Same pending/provisioned flow as provisioning.

**`RenewExpiringCertificates`** – Bulk renewal of certificates expiring before a given threshold. Accepts an `async`
flag.

**`RevokeCertificate`** – Calls the certificate manager's `revoke` method and updates the domain record.

### Events

#### Domain Lifecycle

| Event                | Payload              | When                                |
|----------------------|----------------------|-------------------------------------|
| `DomainCreated`      | Tenancy, Domain      | A new domain is registered          |
| `DomainDeleted`      | Tenancy, Domain      | A domain is removed                 |
| `DomainMarkedPrimary`| Tenancy, Domain      | A domain is set as primary          |

#### Verification

| Event                         | Payload                     | When                              |
|-------------------------------|-----------------------------|-----------------------------------|
| `DomainVerificationSucceeded` | Tenancy, Domain             | TXT record check passed           |
| `DomainVerificationFailed`    | Tenancy, Domain, reason     | TXT record check failed           |
| `DomainUnverified`            | Tenancy, Domain             | Domain marked as unverified       |

#### Validation

| Event                         | Payload                     | When                              |
|-------------------------------|-----------------------------|-----------------------------------|
| `DomainValidationSucceeded`   | Tenancy, Domain             | DNS records match config          |
| `DomainValidationFailed`      | Tenancy, Domain, reason     | DNS records don't match           |
| `DomainInvalidated`           | Tenancy, Domain             | Single domain invalidated         |
| `DomainsInvalidated`          | Tenancy, count              | Bulk invalidation (single query)  |

#### SSL

| Event                    | Payload                          | When                              |
|--------------------------|----------------------------------|-----------------------------------|
| `SslProvisioningStarted` | Tenancy, Domain                 | Provisioning initiated            |
| `SslProvisioned`         | Tenancy, Domain, Certificate    | Certificate obtained              |
| `SslProvisioningFailed`  | Tenancy, Domain, reason         | Provisioning failed               |
| `SslRenewed`             | Tenancy, Domain, Certificate    | Certificate renewed               |
| `SslRenewalFailed`       | Tenancy, Domain, reason         | Renewal failed                    |
| `SslExpiring`            | Tenancy, Domain                 | Certificate approaching expiry    |
| `SslExpired`             | Tenancy, Domain                 | Certificate marked as expired     |
| `SslRevoked`             | Tenancy, Domain                 | Certificate revoked               |

#### Resolution

These events fire on every matching request, regardless of the `resolve` option setting:

| Event                          | Payload              | When                                      |
|--------------------------------|----------------------|-------------------------------------------|
| `UnverifiedDomainResolution`   | Tenancy, Domain      | Resolved a domain that isn't verified     |
| `UnvalidatedDomainResolution`  | Tenancy, Domain      | Resolved a domain that isn't validated    |
| `StaleDomainResolution`        | Tenancy, Domain      | Resolved a domain with a stale checksum   |

### Artisan Commands

#### Domain Management

| Command                  | Description                                            |
|--------------------------|--------------------------------------------------------|
| `sprout:domain:add`      | Add a domain to a tenant (respects single/multiple)   |
| `sprout:domain:remove`   | Remove a domain                                        |
| `sprout:domain:list`     | List domains, optionally filtered by tenancy/tenant    |
| `sprout:domain:primary`  | Set a domain as primary                                |

#### Verification & Validation

| Command                       | Description                                       |
|-------------------------------|---------------------------------------------------|
| `sprout:domain:verify`        | Verify a single domain's TXT record               |
| `sprout:domain:validate`      | Validate a single domain's DNS records             |
| `sprout:domain:reverify`      | Bulk re-check all verified domains (`--async`)     |
| `sprout:domain:revalidate`    | Bulk re-check all validated domains (`--async`)    |
| `sprout:domain:invalidate`    | Bulk reset validation status                       |

#### SSL

| Command                       | Description                                       |
|-------------------------------|---------------------------------------------------|
| `sprout:ssl:provision`        | Provision SSL for a domain                         |
| `sprout:ssl:check`            | Check pending provision/renewal status             |
| `sprout:ssl:renew`            | Renew a single domain's certificate                |
| `sprout:ssl:renew-expiring`   | Bulk renew certificates expiring before threshold  |
| `sprout:ssl:status`           | Show SSL status for domains                        |
| `sprout:ssl:revoke`           | Revoke a certificate                               |

Bulk commands (`reverify`, `revalidate`, `renew-expiring`) accept an `--async` flag to dispatch individual jobs
rather than running inline.

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
- Built-in fallback on the resolver is simpler and sufficient

### Everything in Repository Config

Initially bundled DNS, verification, SSL, and fallback settings into the repository configuration. Rejected because:

- Repository is a storage concern – it shouldn't know about SSL or DNS
- DNS and verification are per-tenancy operational settings, not storage config
- Fallback is a resolution concern, belonging on the resolver
- The auth-style separation (storage vs operational vs resolution) provides cleaner separation of concerns

### Storing Repository Name on Domain Records

Considered tracking which repository a domain belongs to in the database. Rejected because:

- The repository is an infrastructure detail, not a meaningful scope
- Changing repository config would orphan records
- The tenancy name is the meaningful scope and handles all necessary differentiation

## Implementation Plan

### Phase 1: Foundation

1. `Domain` contract
2. `DomainRepository` contract and Eloquent implementation
3. Migration for the domains table
4. `DnsRecordType` enum, `DnsRecord` contract, `DnsLookupDriver` contract and native implementation
5. `VerificationTokenGenerator` contract and random implementation
6. `ActionResult`, `ActionStatus`, and result subclasses

### Phase 2: Core Actions and Resolution

7. `AddDomain` action (respects single/multiple, generates token)
8. `RemoveDomain` action
9. `SetPrimaryDomain` action
10. Domain resolver with fallback, resolve options, checksum staleness checks
11. URL generation with usable domain checks

### Phase 3: Verification and Validation

12. `VerifyDomain` action and job
13. `ValidateDomain` action and job
14. `ReverifyDomains` bulk action (async support)
15. `RevalidateDomains` bulk action (async support)
16. `InvalidateDomains` bulk action
17. Checksum generation and comparison

### Phase 4: SSL

18. `Certificate` contract
19. `CertificateResult` contract
20. `CertificateManager` and `PollableCertificateManager` contracts
21. Let's Encrypt driver
22. Certbot driver
23. Manual driver
24. `ProvisionCertificate` action and job
25. `CheckCertificate` action and job
26. `RenewCertificate` action and job
27. `RenewExpiringCertificates` bulk action
28. `RevokeCertificate` action and job

### Phase 5: CLI and Events

29. All events
30. All artisan commands
31. Config publishing and service provider

### Phase 6: Documentation

32. Configuration guide
33. Setup and usage guide
34. Driver implementation guide for custom drivers

Each phase is independently shippable. Phases 1–3 provide a working domain resolver with verification and validation.
Phase 4 adds SSL. Phase 5 wraps it up with operational tooling.
