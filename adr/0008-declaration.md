---
id: ADR-0008
title: Treat work without a tenant as global or no-tenant only by declaration
status: accepted
created: 2026-10-18
decided: 2026-10-18
depends: [ADR-0005]
updates: []
obsoletes: []
---

# ADR-0008: Treat work without a tenant as global or no-tenant only by declaration

## Context

A [unit of work](../GLOSSARY.md#unit-of-work) can be without an [active tenant](../GLOSSARY.md#active-tenant) for
different reasons, for example:

- it works outside multitenancy entirely, as an administration area or a report across
  [tenant](../GLOSSARY.md#tenant)s does
- it can run with or without a tenant, as a sign-up page on both the application's own domain and a tenant's does
- it was meant to have a tenant and has none by mistake, as a route meant for a [tenancy](../GLOSSARY.md#tenancy)
  but left out of the route group that identifies its tenant does

Without a tenant, the unit of work itself does not show which reason applies. Work outside multitenancy runs in the
[global context](../GLOSSARY.md#global-context), work that can run with or without a tenant runs in the
[no-tenant context](../GLOSSARY.md#no-tenant-context), and some behaviour differs between the two. For example,
where tenants share tables of [tenant data](../GLOSSARY.md#tenant-data), a unit of work in the global context sees
every row, and one in the no-tenant context sees only the rows that belong to no particular tenant. Under
[ADR-0005](0005-mechanisms.md), Sprout refuses an operation whose isolation it cannot establish.

## Decision

We will place a [unit of work](../GLOSSARY.md#unit-of-work) in the [global context](../GLOSSARY.md#global-context)
only when it has made a [declaration](../GLOSSARY.md#declaration) that it works outside multitenancy, and in the
[no-tenant context](../GLOSSARY.md#no-tenant-context), for a [tenancy](../GLOSSARY.md#tenancy) with no
[active tenant](../GLOSSARY.md#active-tenant), only when it has declared that it can run with or without a
[tenant](../GLOSSARY.md#tenant). Where a unit of work has made no declaration and a tenancy has no active tenant,
the unit of work will be in neither the global context nor, for that tenancy, the no-tenant context, and behaviour
that differs between the two, such as an Eloquent query on [tenant data](../GLOSSARY.md#tenant-data), will refuse to
run.

## Alternatives

**Treating work with no declaration as the no-tenant context.** A [unit of work](../GLOSSARY.md#unit-of-work) with
no [tenant](../GLOSSARY.md#tenant) and no [declaration](../GLOSSARY.md#declaration) would run as though it could run
without one. We rejected this because a unit of work missing its tenant by mistake would still run, for example,
where tenants share tables, seeing only the rows that belong to no particular tenant, with nothing to show that a
tenant was missing.

**Treating work with no declaration as the global context.** A unit of work with no tenant and no declaration would
run as though it worked outside multitenancy. We rejected this because a unit of work missing its tenant by mistake
would, for example, see every row where tenants share tables.

**Letting the application decide where work with no declaration runs.** A setting in the application's configuration
would choose the [global context](../GLOSSARY.md#global-context) or the
[no-tenant context](../GLOSSARY.md#no-tenant-context) for the work with no declaration that it covers. We rejected
this because it would bring back whichever of the two alternatives above the setting names, including for work
missing its tenant by mistake.

**Refusing work with no declaration at its start.** A unit of work with no tenant and no declaration would be
refused before it ran, whatever it went on to use. We rejected this because work that has nothing to do with
multitenancy, such as a health check or a framework command, would have to make a declaration before it could run,
and Sprout would get in the way of work it plays no part in.

## Consequences

Easier:

- A [unit of work](../GLOSSARY.md#unit-of-work) missing its [tenant](../GLOSSARY.md#tenant) by mistake will fail
  where it first uses behaviour that differs between the [global context](../GLOSSARY.md#global-context) and the
  [no-tenant context](../GLOSSARY.md#no-tenant-context).
- Work that never uses [tenant data](../GLOSSARY.md#tenant-data), such as a health check, will run without a
  [declaration](../GLOSSARY.md#declaration).

Harder:

- Every unit of work that uses tenant data without a tenant, such as a console command, a queued job dispatched with
  no tenant, or a test, will have to make a declaration.
- An application that adopts Sprout will have to declare existing work that uses tenant data without a tenant, with
  no setting to do it for all of that work at once.

Constrained:

- Every type of unit of work, including one an application adds, will need a way to make a declaration.
- Sprout will not place a unit of work that has made no declaration in the global context or the no-tenant context,
  and no setting will change that.
