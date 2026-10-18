---
id: ADR-0006
title: Make a tenant active only by entering it
status: accepted
created: 2026-10-18
decided: 2026-10-18
depends: []
updates: []
obsoletes: []
---

# ADR-0006: Make a tenant active only by entering it

## Context

A [tenant](../GLOSSARY.md#tenant) can be found in several ways, such as from a request, from a queued job that was
dispatched for it, or by application code in a console command or a test. Applications can add their own.

Making a tenant the [active tenant](../GLOSSARY.md#active-tenant) of its [tenancy](../GLOSSARY.md#tenancy) is
governed by rules. For example, [ADR-0001](0001-tenant-switching.md) allows it only when the tenancy has no active
tenant, or for a [child tenant](../GLOSSARY.md#child-tenant) beneath its active parent, and the application may
refuse a tenant, such as one that is suspended. It also involves work, such as setting up everything that depends on
the tenant.

## Decision

We will make a [tenant](../GLOSSARY.md#tenant) active through one operation, [entering](../GLOSSARY.md#entering),
and through nothing else. Entering applies every rule about making a tenant active, such as those of
[ADR-0001](0001-tenant-switching.md) and the application's refusal, and does the work that goes with it, such as
setting up what depends on the tenant. Ways of finding a tenant, such as
[identification](../GLOSSARY.md#identification), [restoration](../GLOSSARY.md#restoration) and application code that
already has one, only find or supply it and hand it to entering.

## Alternatives

**Each way of finding a tenant making it active.** [Identification](../GLOSSARY.md#identification) and
[restoration](../GLOSSARY.md#restoration) would each set the [active tenant](../GLOSSARY.md#active-tenant) and apply
the rules themselves. We rejected this because the rules would be repeated in each, and a way that misses one,
including a way the application adds, would make a [tenant](../GLOSSARY.md#tenant) active without it.

**A method that sets the active tenant directly, alongside entering.** [Entering](../GLOSSARY.md#entering) would
apply the rules, and the method would set the tenant without them, for code that needs to skip one. We rejected this
because the method skips every rule at once, including those of [ADR-0001](0001-tenant-switching.md), for code that
needed to skip only one.

## Consequences

Easier:

- A rule about making a [tenant](../GLOSSARY.md#tenant) active will be written and tested in one place.
- A way of finding a tenant that the application adds, such as its own way of reading a request, will have every rule
  applied without applying any itself.

Harder:

- Work that must reach a tenant the application would refuse, such as migrating the database of a suspended tenant,
  will need a form of [entering](../GLOSSARY.md#entering) that allows it, rather than a way around entering.

Constrained:

- [Identification](../GLOSSARY.md#identification) and [restoration](../GLOSSARY.md#restoration) will not set the
  [active tenant](../GLOSSARY.md#active-tenant) themselves.
- Sprout will have no way to make a tenant active other than entering.
