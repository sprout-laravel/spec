# Sprout Specification

This repository contains the technical specification, architecture documentation, and design decisions for Sprout.

## Contents

### [Architecture](./architecture/)

Technical documentation describing how Sprout works internally. This includes component diagrams, data flows, and
explanations of core concepts.

- [Overview](./architecture/overview.md) — High-level architecture and design principles
- [Resolution Hooks](./architecture/resolution-hooks.md) — How and when Sprout resolves tenant identities
- [Tenancy Lifecycle](./architecture/tenancy-lifecycle.md) — How tenants are identified, resolved, and managed
- [Service Overrides](./architecture/service-overrides.md) — How Sprout integrates with Laravel's service container
- [Components](./architecture/components/) — Detailed documentation for each component (Core, Bud, Seedling, Canopy)

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
