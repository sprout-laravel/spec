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

### Interactive Configuration Commands

Propagator provides interactive terminal commands for managing Sprout configuration files, offering a user-friendly UI instead of manual file editing.

#### `sprout:configure`

An interactive command for managing Sprout's configuration. Guides developers through setting up tenancies, providers, resolvers, and overrides with prompts and validation.

```bash
php artisan sprout:configure
php artisan sprout:configure tenancy
php artisan sprout:configure provider
```

The interactive UI presents available options, validates input, and writes changes to the appropriate configuration files.

### Extension API

Propagator exposes an API that allows other Sprout packages to integrate dynamically. This enables packages to:

- Register additional configuration options for the interactive commands
- Provide custom make command stubs
- Add package-specific entries to `sprout:list` output
- Extend the configuration UI with their own settings

For example, Seedling could register its database-related tenancy options so they appear in `sprout:configure tenancy`, and Canopy could add domain configuration options.

```php
// Example: A package registering options with Propagator
Propagator::extend('tenancy', function (TenancyConfigurator $configurator) {
    $configurator->addOption('database', 'Configure tenant database connection');
});
```

### Telescope Integration

When Laravel Telescope is detected, Propagator integrates with it to make the various collectors tenant-aware. This allows developers to filter and view Telescope entries by tenant, providing better visibility into tenant-specific behaviour during development.

The integration hooks into Telescope's watchers to:

- Tag entries with the current tenant identifier
- Add tenant context to request, job, query, and other recorded entries
- Enable filtering the Telescope UI by tenant

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
- What should the extension API contracts look like?
- How should packages register their configuration options with Propagator?

## Implementation Plan

1. Create separate repository with package structure and service provider
2. Define the extension API contracts
3. Implement make commands with stub files
4. Implement `sprout:list` command
5. Implement interactive configuration commands
6. Implement Telescope integration (conditional on Telescope being installed)
