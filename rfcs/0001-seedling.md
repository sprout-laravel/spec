# RFC-0001: Seedling — Multi-Database Support

- **Status:** Draft
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-04
- **Updated:** 2026-02-04

## Summary

Seedling provides multi-tenant database support for Sprout, enabling tenants to have isolated database environments. It
supports multiple isolation strategies, configurable migration paths, and explicit provisioning workflows, while
integrating with Bud for tenant-specific database configuration.

## Motivation

Many multi-tenant applications require database-level isolation between tenants. The reasons vary:

- **Data security** — Regulatory or contractual requirements mandate physical separation
- **Performance** — Large tenants benefit from dedicated resources
- **Customisation** — Tenants may need schema modifications or different database engines
- **Backup/restore** — Per-tenant backup and restore operations are simpler with isolation

Sprout Core provides tenant context and scoping for shared-database scenarios (where all tenants share tables with a
`tenant_id` column), but this doesn't address cases where tenants need their own database or schema.

Seedling fills this gap by providing:

1. Support for multiple database isolation strategies
2. Tenant-scoped migration and seeding tooling
3. Provisioning and deprovisioning workflows
4. Integration with Bud for dynamic database configuration

### Goals

- Support all common isolation strategies without forcing a particular approach
- Provide a clean migration story that feels like Laravel's native tooling
- Make provisioning explicit and controllable, not magic
- Integrate cleanly with Bud's configuration system
- Remain flexible enough to support edge cases

### Non-Goals

- Seedling does not handle shared-database tenancy (Core already does this via scoping)
- Seedling does not manage database servers themselves (creating MySQL instances, etc.)
- Seedling does not handle cross-tenant queries or data aggregation

## Detailed Design

### Isolation Strategies

Seedling supports three isolation strategies:

**Separate Database** — Each tenant has their own database on a shared or dedicated server:

```
mysql-server/
├── app_central
├── tenant_acme
├── tenant_globex
└── tenant_initech
```

**Separate Schema** — Tenants share a database but have isolated schemas (PostgreSQL):

```
postgres-database/
├── public          (central)
├── tenant_acme
├── tenant_globex
└── tenant_initech
```

**Separate Connection** — Each tenant has entirely separate connection details, potentially on different servers or even
different database engines:

```
acme     → mysql://acme-db.example.com/acme
globex   → pgsql://globex-db.example.com/main
initech  → mysql://shared.example.com/initech
```

The strategy is configuration-based and can potentially vary per tenant when using Bud's external configuration.

### Configuration

```php
// config/sprout.php
return [
    'seedling' => [
        // The base connection to use as a template
        'connection' => 'tenant',
        
        // Default isolation strategy
        'strategy' => 'database', // 'database', 'schema', 'connection'
        
        // How to derive the database/schema name from tenant
        'database' => [
            'prefix' => 'tenant_',
            'suffix' => '',
            // Or use a callback/class for full control
            'resolver' => null,
        ],
        
        // Migration configuration
        'migrations' => [
            'path' => database_path('migrations/tenant'),
            'table' => 'migrations',
        ],
        
        // Seeder configuration  
        'seeders' => [
            TenantDatabaseSeeder::class,
        ],
    ],
];
```

### Integration with Bud

Seedling uses Bud's service override system but extends it for database-specific needs. The flow:

1. Tenant is activated
2. Bud loads tenant configuration (from model, file, database, etc.)
3. Seedling's override applies the database configuration to the `tenant` connection
4. Application code using the `tenant` connection now hits the tenant's database

For simple cases (derive database name from tenant key), no Bud configuration is needed — Seedling handles it directly.
For complex cases (per-tenant connection strings, credentials, etc.), Bud provides the configuration source.

```php
// Simple: database name derived from tenant
// No additional config needed, Seedling uses prefix + tenant_key

// Complex: full connection details from Bud
// Tenant config (in database, file, etc.):
[
    'database' => [
        'host' => 'acme-db.example.com',
        'database' => 'acme_production',
        'username' => 'acme_user',
        'password' => '...',
    ],
]
```

### Connection Management

Seedling provides a `tenant` database connection that dynamically resolves to the current tenant's database:

```php
// In config/database.php
'connections' => [
    'tenant' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'database' => null, // Filled by Seedling at runtime
        // ... other defaults
    ],
],
```

Models specify this connection:

```php
class TenantUser extends Model
{
    protected $connection = 'tenant';
}
```

When no tenant is active, accessing the `tenant` connection throws an exception (configurable behaviour).

### Migrations

Tenant migrations live in a configurable directory (default: `database/migrations/tenant`). Seedling provides Artisan
commands that mirror Laravel's:

```bash
# Run migrations for all tenants
php artisan sprout:migrate

# Run for specific tenant
php artisan sprout:migrate --tenant=acme

# Run for tenants matching criteria
php artisan sprout:migrate --where="plan=premium"

# Other migration commands
php artisan sprout:migrate:rollback
php artisan sprout:migrate:fresh
php artisan sprout:migrate:status
```

Each command:

1. Iterates over tenants (or specified tenant)
2. Activates the tenant context
3. Runs the migration against the tenant's database
4. Deactivates and moves to the next

Migration state is tracked per-tenant in each tenant's database.

### Provisioning

Provisioning is explicit — Seedling does not automatically create databases when tenants are created. This is
intentional:

- Tenant models are user-defined; Seedling can't assume lifecycle hooks
- Provisioning may require additional setup beyond database creation
- Some applications provision asynchronously or with approval workflows

Provisioning is triggered via:

```php
// Programmatically
Sprout::provision($tenant);

// Or via Artisan
php artisan sprout:provision acme
```

The provisioning process:

1. Creates the database/schema (strategy-dependent)
2. Runs all tenant migrations
3. Runs tenant seeders (if configured)
4. Dispatches `TenantProvisioned` event

Deprovisioning follows a similar pattern:

```php
Sprout::deprovision($tenant);
```

Users wire these into their tenant lifecycle as appropriate:

```php
// In a controller, job, or event listener
public function store(Request $request)
{
    $tenant = Tenant::create($request->validated());
    
    // Immediate provisioning
    Sprout::provision($tenant);
    
    // Or dispatch a job
    ProvisionTenant::dispatch($tenant);
}
```

### Events

| Event                  | When                                   |
|------------------------|----------------------------------------|
| `TenantProvisioning`   | Before provisioning begins             |
| `TenantProvisioned`    | After provisioning completes           |
| `TenantDeprovisioning` | Before deprovisioning begins           |
| `TenantDeprovisioned`  | After deprovisioning completes         |
| `TenantMigrating`      | Before migrations run for a tenant     |
| `TenantMigrated`       | After migrations complete for a tenant |

## Open Questions

### Connection Purging

When switching between tenants (in commands that iterate), how do we ensure connections are properly closed and
reopened? Laravel's `DB::purge()` exists but there may be edge cases with persistent connections or connection pooling.

### Schema Strategy Implementation

For PostgreSQL schema isolation, what's the cleanest way to switch schemas? Options include:

- Setting `search_path` on the connection
- Using schema-qualified table names
- Separate connections per schema

### Migration Isolation

Should tenant migrations be completely isolated from central migrations, or should there be a way to share/inherit? For
example, a `users` table that exists in both central and tenant databases with the same structure.

### Parallel Provisioning

For commands that operate on many tenants, should there be built-in support for parallel execution? This significantly
speeds up operations but adds complexity around connection management and output handling.

### Failure Handling

What happens when provisioning fails partway through (database created, but migrations failed)? Options:

- Leave in partial state, let user fix manually
- Attempt rollback (delete database)
- Mark tenant as "provisioning failed" with state tracking

### Testing

How should tenant database testing work? Options:

- SQLite in-memory per tenant (fast but different from production)
- Transaction wrapping (complex with separate databases)
- Actual test databases (slow but accurate)

## Alternatives Considered

### Automatic Provisioning via Model Events

Rejected because tenant models are user-defined and the provisioning lifecycle varies significantly between
applications. Some need synchronous provisioning, others async, others need approval workflows.

### Single Migration Directory with Tenant Flag

Could use Laravel's existing migration directory with a `--tenant` flag or trait to mark tenant migrations. Rejected
because it conflates central and tenant migrations, making it harder to reason about what runs where.

### Connection-per-Tenant in Config

Could define each tenant's connection statically in `config/database.php`. Rejected because it doesn't scale and
requires config changes for each new tenant.

## Implementation Plan

1. **Connection management** — Dynamic connection configuration based on active tenant
2. **Migration tooling** — Artisan commands for tenant migrations
3. **Provisioning** — Database/schema creation and migration running
4. **Bud integration** — Support for external configuration sources
5. **Testing utilities** — Helpers for testing tenant database code
