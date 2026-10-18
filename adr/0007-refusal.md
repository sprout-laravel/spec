---
id: ADR-0007
title: Bypass the application's refusal only explicitly
status: accepted
created: 2026-10-18
decided: 2026-10-18
depends: [ADR-0005, ADR-0006]
updates: [ADR-0006]
obsoletes: []
---

# ADR-0007: Bypass the application's refusal only explicitly

## Context

Which tenants may be served is the application's rule ([ADR-0005](0005-mechanisms.md)), and an application may not
want to serve every [tenant](../GLOSSARY.md#tenant) it has, such as one that is suspended or has not paid. A tenant
can be found in several ways, such as through [identification](../GLOSSARY.md#identification) of a request, through
[restoration](../GLOSSARY.md#restoration) for a queued job, or by application code, and each of them hands the
tenant to [entering](../GLOSSARY.md#entering), which applies the application's refusal
([ADR-0006](0006-entering.md)).

Some work has to reach a tenant the application would refuse, such as an administration area working with a
suspended tenant, or keeping a suspended tenant's database migrated. Some of that work is Sprout's own, such as
migrating every tenant's database. An application may also want to know when a tenant is entered, such as to record
access to tenants it would refuse.

## Decision

We will let work reach a [tenant](../GLOSSARY.md#tenant) the application would refuse only through
[unrefusable entering](../GLOSSARY.md#unrefusable-entering), a separate form of [entering](../GLOSSARY.md#entering)
that code will have to ask for explicitly, and that no ordinary call to enter a tenant will produce. The application
will be told about every entering, including whether it is unrefusable, and will be able to refuse any entering that
is not. A refused entering will fail, and the tenant will not become active. Unrefusable entering will remove only
the application's refusal: it will do the same work as any other entering, and every other rule of entering, such as
those of [ADR-0001](0001-tenant-switching.md), will still apply.

## Alternatives

**Every entering refusable, with no bypass.** Work that must reach a [tenant](../GLOSSARY.md#tenant) the application
would refuse would have to get past the application's own rule, such as by setting a flag the rule checks. We
rejected this because Sprout's own work, such as migrating every tenant's database, cannot get past a rule the
application wrote, and each application would build its own way around its rule, which would look like any other
[entering](../GLOSSARY.md#entering).

**A bypass the application is not told about.** [Unrefusable entering](../GLOSSARY.md#unrefusable-entering) would
skip the application entirely. We rejected this because the application would not know that a tenant it would refuse
had been entered, and could not, for example, record it.

## Consequences

Easier:

- Work that reaches a [tenant](../GLOSSARY.md#tenant) the application would refuse, such as migrating a suspended
  tenant's database, will be visible in the code and will be searchable.
- The application will be able to record every [entering](../GLOSSARY.md#entering), including those it cannot refuse.

Harder:

- When a queued job's tenant is entered ordinarily and the application refuses it, the entering will fail, and what
  happens to the job, such as discarding it or retrying it later, will be the application's to decide.
- The response to a request whose entering is refused will have no [active tenant](../GLOSSARY.md#active-tenant). The
  application will be able to answer it from the tenant's own details without entering it, but anything in it that
  depends on the tenant being active, such as the tenant's own files, will need
  [unrefusable entering](../GLOSSARY.md#unrefusable-entering).
- Any code will be able to use unrefusable entering to get past the application's rule. It will be searchable, but not
  prevented.

Constrained:

- The application will be told about every entering, whatever its form.
- Unrefusable entering will remove only the application's refusal, never any other rule of entering.
