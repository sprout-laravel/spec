# Architecture

This directory contains technical architecture documentation for Sprout.

## What is Architecture Documentation?

Architecture documents explain how Sprout works internally. They describe the design decisions, component interactions,
and implementation details that make up the system.

## Index

| Document                                       | Description                                   | Status   |
|------------------------------------------------|-----------------------------------------------|----------|
| [Overview](./overview.md)                      | High-level architecture and design principles | Complete |
| [Resolution Hooks](./resolution-hooks.md)      | When and how tenant resolution occurs         | Complete |
| [Identity Resolvers](./identity-resolvers.md)  | Extracting tenant identity from requests      | Complete |
| [Tenancy Lifecycle](./tenancy-lifecycle.md)    | Events and listeners during tenant activation | Complete |
| [Service Overrides](./service-overrides.md)    | Making Laravel services tenant-aware          | Stub     |

### Component Documentation

Detailed documentation for each Sprout component:

- [Core](./components/core/) — Foundation package
- [Bud](./components/bud/) — Tenant-specific configuration
- [Seedling](./components/seedling/) — Multi-database support
- [Canopy](./components/canopy/) — Domain-based identification
