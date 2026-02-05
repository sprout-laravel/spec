# Architecture Overview

This document provides a high-level overview of Sprout's architecture and design principles.

## Design Principles

Sprout is built around several core principles:

### Explicit Over Magic

Sprout avoids hidden behaviour and "magic" that can make debugging difficult. Configuration is explicit, and the flow of
tenant resolution and context management is predictable and traceable.

### Seamless Integration

Sprout integrates with Laravel's existing patterns rather than fighting against them. It uses service providers,
middleware, and the service container in the way Laravel intends, making it feel like a natural extension of the
framework.

### Flexibility Without Compromise

The package supports a wide range of multitenancy patterns without forcing users into a specific approach. Whether you
need subdomain-based tenancy, path-based identification, or custom resolution logic, Sprout accommodates your needs.

### Separation of Concerns

Each component has a clear responsibility and well-defined boundaries. This makes the codebase easier to understand,
test, and extend.

## Package Overview

Sprout v2 is organised into four packages within a monorepo:

```
┌─────────────────────────────────────────────────────────────────┐
│                           Sprout                                │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│      Core       │       Bud       │    Seedling     │  Canopy   │
├─────────────────┼─────────────────┼─────────────────┼───────────┤
│ • Tenancy       │ • Tenant config │ • Multi-database│ • Domains │
│ • Resolution    │ • Driver config │ • Migrations    │ • SSL     │
│ • Overrides     │                 │ • Provisioning  │           │
│ • Eloquent      │                 │                 │           │
└─────────────────┴─────────────────┴─────────────────┴───────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │     Laravel     │
                    └─────────────────┘
```

### Core

The foundation that all other packages build on. Handles:

- [Tenancy](./tenancy.md) — Managing tenant state
- [Identity resolution](./identity-resolvers.md) — Extracting tenant identity from requests
- [Service overrides](./service-overrides.md) — Making Laravel services tenant-aware
- [Eloquent integration](./eloquent-integration.md) — Tenant-aware models

### Bud

Manages tenant-specific configuration for Laravel's driver-based services. Lets each tenant have their own mail, cache,
filesystem, or queue settings.

### Seedling

Handles tenant-specific databases. Provides separate database connections per tenant, tenant-scoped migrations, and
database provisioning.

### Canopy

Provides domain-based tenant identification. Supports custom domains per tenant, SSL certificate management, and domain
verification.

## Request Lifecycle

The following diagram shows how a request flows through Sprout:

```
Request
   │
   ▼
┌──────────────────────────────────┐
│         Route Matching           │
│  (Routing hook: resolve early)   │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│      Tenancy Middleware          │
│ (Middleware hook: resolve here)  │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│       Tenant Resolution          │
│   (Load tenant from identity)    │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│      Context Activation          │
│   • Service overrides applied    │
│   • Events dispatched            │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│        Controller/Action         │
│   (Tenant context available)     │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│      Context Deactivation        │
│   • Service overrides reverted   │
│   • Events dispatched            │
└──────────────────────────────────┘
   │
   ▼
Response
```

Resolution can happen at two points — the [routing hook or the middleware hook](./resolution-hooks.md). The routing hook
fires earlier (before middleware), which is useful when service overrides need to configure services before middleware
runs. The middleware hook fires later, which is required for session-based resolution.

See [Tenancy Lifecycle](./tenancy-lifecycle.md) for details on what happens after resolution.

## Extension Points

Sprout uses a [driver-based factory pattern](./managers-factories.md) that enables extension without modifying core
code:

- **Custom Identity Resolvers** — Register new ways to identify tenants from requests
- **Custom Tenant Providers** — Load tenants from any data source
- **Custom Service Overrides** — Make additional Laravel services tenant-aware
- **Event Listeners** — React to [tenancy lifecycle events](./tenancy-lifecycle.md)

## Further Reading

- [Resolution Hooks](./resolution-hooks.md) — When tenant resolution occurs
- [Tenancy Lifecycle](./tenancy-lifecycle.md) — Events during tenant activation
- [Managers & Factories](./managers-factories.md) — The extensibility pattern
