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

## Component Overview

Sprout v2 consolidates functionality into a single package with four logical components:

```
┌─────────────────────────────────────────────────────────────────┐
│                           Sprout                                │
├─────────────────┬─────────────────┬─────────────────┬───────────┤
│      Core       │       Bud       │    Seedling     │  Canopy   │
├─────────────────┼─────────────────┼─────────────────┼───────────┤
│ • Tenancy       │ • Configuration │ • Database      │ • Domain  │
│ • Resolution    │ • Services      │ • Migrations    │ • Routing │
│ • Context       │ • Connections   │ • Seeding       │ • SSL     │
│ • Events        │ • Overrides     │                 │           │
└─────────────────┴─────────────────┴─────────────────┴───────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │     Laravel     │
                    └─────────────────┘
```

### Core

The foundation of Sprout. Handles tenant identification, resolution, and context management. All other components build
on top of Core.

See [Components: Core](./components/core/) for details.

### Bud

Manages tenant-specific configuration. Provides a clean way to override Laravel's driver-based services (mail, cache,
filesystem, etc.) on a per-tenant basis.

See [Components: Bud](./components/bud/) for details.

### Seedling

Handles tenant-specific databases. Includes support for separate database connections per tenant, tenant-scoped
migrations, and database provisioning.

See [Components: Seedling](./components/seedling/) for details.

### Canopy

Provides domain-based tenant identification. Includes support for custom domains, SSL certificate management, and domain
verification.

See [Components: Canopy](./components/canopy/) for details.

## Request Lifecycle

The following diagram shows how a request flows through Sprout:

```
Request
   │
   ▼
┌──────────────────────────────────┐
│         Route Matching           │
│   (Identity resolved if eager)   │
└──────────────────────────────────┘
   │
   ▼
┌──────────────────────────────────┐
│      Tenancy Middleware          │
│   (Identity resolved if lazy)    │
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
│   • Configuration loaded         │
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

See [Tenancy Lifecycle](./tenancy-lifecycle.md) for a detailed explanation.

## Extension Points

Sprout provides several extension points for customisation:

- **Custom Identity Resolvers** — Implement your own tenant identification logic
- **Custom Tenant Providers** — Load tenants from any data source
- **Service Override Drivers** — Add support for additional Laravel services
- **Event Listeners** — React to tenancy lifecycle events

## Further Reading

- [Tenancy Lifecycle](./tenancy-lifecycle.md)
- [Service Overrides](./service-overrides.md)
- [Component Documentation](./components/)
