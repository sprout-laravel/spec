# RFC-0004: Propagator — Developer Toolkit

- **Status:** Draft
- **Author:** Ollie Read (@ollieread)
- **Created:** 2026-02-05
- **Updated:** 2026-02-05

## Summary

Propagator is a developer toolkit for Sprout that provides Artisan commands for code generation, inspection, and other development workflows. It simplifies working with Sprout by offering `make:*` commands for generating tenant-aware classes and inspection commands for understanding the current configuration.

## Motivation

Working with Sprout requires creating various classes that follow specific contracts and patterns: tenant models, tenant providers, identity resolvers, and service overrides. Currently, developers must create these manually, referencing documentation and existing implementations.

A dedicated dev toolkit addresses several pain points:

1. **Boilerplate reduction** — Generating stubs for common Sprout classes saves time and ensures correct structure
2. **Discoverability** — Inspection commands help developers understand what's registered and configured
3. **Consistency** — Generated code follows established patterns and conventions
4. **Onboarding** — New developers can scaffold components without deep knowledge of internals

## Detailed Design

Propagator is a separate package in its own repository, installed as a dev dependency.

### Make Commands

#### `make:tenant`

Generates a tenant model class implementing the `Tenant` contract.

```bash
php artisan make:tenant Organisation
```

#### `make:provider`

Generates a tenant provider class implementing the `TenantProvider` contract.

```bash
php artisan make:provider OrganisationProvider
```

#### `make:resolver`

Generates an identity resolver class implementing the `IdentityResolver` contract.

```bash
php artisan make:resolver CustomResolver
```

#### `make:override`

Generates a service override class implementing the `ServiceOverride` contract.

```bash
php artisan make:override CustomCacheOverride
```

### Inspection Commands

#### `sprout:list`

Lists registered tenancies, providers, resolvers, and overrides.

```bash
php artisan sprout:list
php artisan sprout:list --tenancies
php artisan sprout:list --providers
php artisan sprout:list --resolvers
php artisan sprout:list --overrides
```

## Drawbacks

- **Additional package** — Another package to maintain, even if lightweight
- **Laravel coupling** — Artisan commands tie this specifically to Laravel (though Sprout is Laravel-focused anyway)

## Alternatives

- **No toolkit** — Developers create classes manually using documentation
- **Built into Core** — Include commands in the core package (rejected: dev tools shouldn't be a production dependency)

## Unresolved Questions

- Should Propagator include testing utilities (factories, assertions, test traits)?
- What additional inspection commands would be valuable?
- Should make commands support customisation options (e.g., `--eloquent` for providers)?

## Implementation Plan

1. Create separate repository with package structure and service provider
2. Implement make commands with stub files
3. Implement `sprout:list` command
