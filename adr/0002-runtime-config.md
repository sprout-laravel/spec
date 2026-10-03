---
id: ADR-0002
title: Never rewrite configuration at runtime
status: proposed
created: 2026-10-03
decided:
depends: []
updates: []
obsoletes: []
---

# ADR-0002: Never rewrite configuration at runtime

## Context

A [tenant](../GLOSSARY.md#tenant) can have its own
[per-tenant configuration](../GLOSSARY.md#per-tenant-configuration), such as its own database connection, mailer or
storage disk, read from wherever the application keeps it, such as a database table, files or an external secret
store.

Laravel builds services from configuration when they are first used, and keeps what it builds. Its managers hold on
to the connections, mailers and cache stores they have resolved, and objects built from them keep the instances they
were given: the rate limiter, for example, receives its cache repository when it is constructed. Changing a
configuration value after a service has been resolved does not reach that service, and whether a change made earlier
takes effect depends on what has already been resolved. In a long-running process such as a queue worker, a changed
value also stays in place for every later [unit of work](../GLOSSARY.md#unit-of-work) unless something puts it back.

## Decision

We will never apply [per-tenant configuration](../GLOSSARY.md#per-tenant-configuration) by changing the
application's configuration at runtime. Each type of per-tenant configuration, such as a
[tenant](../GLOSSARY.md#tenant)'s database connection or mailer, will be provided by a driver that hooks into the
service's own manager, so that the service retrieves the tenant's values and builds from them at the point it needs
them.

## Alternatives

**Writing the tenant's values into the configuration when the tenant becomes active.** The tenant's connection
details would be written into `database.connections`, for example, and Laravel would read them like any other
configuration. We rejected this because services that have already been resolved keep the values they were built
with, the result depends on what happened to be resolved before the [tenant](../GLOSSARY.md#tenant) became active,
and the change has to be undone at exactly the right moment when the tenant is left, or it carries over into the
next [unit of work](../GLOSSARY.md#unit-of-work).

## Consequences

Easier:

- A [tenant](../GLOSSARY.md#tenant)'s values will be applied at a known point, however much has been resolved
  before.
- Leaving a tenant will not depend on putting configuration back.

Harder:

- Each type of [per-tenant configuration](../GLOSSARY.md#per-tenant-configuration) will need a driver written for
  its service, including types that applications register themselves.
- Services whose managers have no way to add a driver, such as Laravel's password broker manager, which has no
  `extend()` method, will take more work to support.

Constrained:

- Sprout will never write per-tenant values into the application's configuration.
