This repository contains the technical specification, architecture documentation, and design decisions for Sprout.

## Contents

### [Architecture](./architecture/)

Technical documentation describing how Sprout works internally. These are design specification documents that explain
concepts, design rationale, and how parts of the system relate to each other.

| Document                                                   | Description                                   |
|------------------------------------------------------------|-----------------------------------------------|
| [Overview](./architecture/overview.md)                     | High-level architecture and design principles |
| [Tenancy](./architecture/tenancy.md)                       | The container for tenant state                |
| [Tenant Providers](./architecture/tenant-providers.md)     | Loading tenants from storage                  |
| [Identity Resolvers](./architecture/identity-resolvers.md) | Extracting tenant identity from requests      |
| [Resolution Hooks](./architecture/resolution-hooks.md)     | When and how tenant resolution occurs         |
| [Tenancy Lifecycle](./architecture/tenancy-lifecycle.md)   | Events and listeners during tenant activation |
| [Service Overrides](./architecture/service-overrides.md)   | Making Laravel services tenant-aware          |
| [Eloquent Integration](./architecture/eloquent-integration.md) | Tenant-aware models and automatic scoping |
| [Managers & Factories](./architecture/managers-factories.md) | Driver-based factory pattern for extensibility |
| [Exceptions](./architecture/exceptions.md)                     | Error handling and exception hierarchy |

#### Extension Packages

The documents above cover Sprout's core architecture. The following extension packages add additional capabilities:

- [Bud](./architecture/components/bud.md) — Tenant-specific configuration
- [Seedling](./architecture/components/seedling/) — Multi-database support
- [Canopy](./architecture/components/canopy/) — Domain-based identification

### [RFCs](./rfcs/)

Request for Comments documents for proposing significant changes or new features. RFCs provide a structured way to
discuss and evaluate proposals before implementation.

- [RFC-0000: Template](./rfcs/0000-template.md) — Template for new RFCs
- [RFC-0001: Seedling — Multi-Database Support](./rfcs/0001-seedling.md) — Multi-database support
- [RFC-0002: Canopy — Domain-Based Tenant Identification](./rfcs/0002-canopy.md) — Domain-based tenant identification
- [RFC-0003: Stacked Identity Resolution](./rfcs/0003-stacked-identity-resolution.md) — Stacked Identity Resolution
- [Index](./rfcs/README.md) — List of all RFCs and their status

### [ADRs](./adrs/)

Architecture Decision Records document significant technical decisions, their context, and rationale. ADRs help maintain
a historical record of why things are built the way they are.

- [ADR-0000: Template](./adrs/0000-template.md) — Template for new ADRs
- [ADR-0001: Monorepo v2](./adrs/0001-monorepo-v2.md) — Decision to consolidate addons into a single package
- [Index](./adrs/README.md) — List of all ADRs

## Contributing

If you'd like to propose a change to Sprout's architecture or design, please:

1. For new features or significant changes, open an RFC
2. For implementation decisions within an accepted RFC, document them as ADRs
3. For clarifications or corrections to existing documentation, open a pull request

See [CONTRIBUTING.md](https://github.com/sprout-laravel/.github/blob/main/CONTRIBUTING.md) for general contribution
guidelines.

## Versioning

This specification tracks the development of Sprout v2. Documentation for v1 can be found in
the [main documentation](https://sprout.ollieread.com).
