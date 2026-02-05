# Architecture

This directory contains technical architecture documentation for Sprout.

## What is Architecture Documentation?

Architecture documents explain how Sprout works internally. They describe the design decisions, component interactions,
and implementation details that make up the system.

## Index

| Document                                       | Description                                   | Status   |
|------------------------------------------------|-----------------------------------------------|----------|
| [Overview](./overview.md)                      | High-level architecture and design principles | Complete |
| [Tenancy](./tenancy.md)                        | The container for tenant state                | Complete |
| [Tenant Providers](./tenant-providers.md)      | Loading tenants from storage                  | Complete |
| [Identity Resolvers](./identity-resolvers.md)  | Extracting tenant identity from requests      | Complete |
| [Resolution Hooks](./resolution-hooks.md)      | When and how tenant resolution occurs         | Complete |
| [Tenancy Lifecycle](./tenancy-lifecycle.md)    | Events and listeners during tenant activation | Complete |
| [Service Overrides](./service-overrides.md)    | Making Laravel services tenant-aware          | Complete |
| [Eloquent Integration](./eloquent-integration.md) | Tenant-aware models and automatic scoping  | Complete |
| [Middleware & Routing](./middleware-routing.md) | Route macros, middleware, and request flow    | Complete |

### Component Documentation

Detailed documentation for each Sprout component:

- [Core](./components/core/) — Foundation package
- [Bud](./components/bud/) — Tenant-specific configuration
- [Seedling](./components/seedling/) — Multi-database support
- [Canopy](./components/canopy/) — Domain-based identification
