---
id: ADR-0009
title: Refuse stale state and stop the process that finds it
status: accepted
created: 2026-10-18
decided: 2026-10-18
depends: [ADR-0001, ADR-0005]
updates: []
obsoletes: []
---

# ADR-0009: Refuse stale state and stop the process that finds it

## Context

In a long-running process, such as an Octane worker or a queue worker, one
[unit of work](../GLOSSARY.md#unit-of-work) follows another, and anything one changes and does not put back is still
there when the next starts. Units of work can also run inside one another, such as a test that makes requests, or a
queued job run immediately inside a request. Several [tenant](../GLOSSARY.md#tenant)s can be active at once, at most
one for each [tenancy](../GLOSSARY.md#tenancy), and under [ADR-0001](0001-tenant-switching.md) a
[child tenant](../GLOSSARY.md#child-tenant) is [entered](../GLOSSARY.md#entering) beneath its active parent. A
tenant entered during a unit of work, for example through [identification](../GLOSSARY.md#identification) of a
request, can still be active when it finishes.

Under [ADR-0001](0001-tenant-switching.md), [leaving](../GLOSSARY.md#leaving) a tenant puts things back as they
were, even if something goes wrong in the code that ran for it. Putting things back can itself fail part of the way
through, for example when restoring one of the services the tenant changed throws an exception. What a failed
cleanup did not put back is not known, and a reset after it cannot be trusted to be complete.

A tenant from a unit of work that has finished can still be active because nothing left it, or still be partly in
place because leaving it failed. A unit of work that starts in either case can run with some of that tenant's state,
such as its database connection or cache prefix. Nothing else stops it: one that enters no tenant, such as a request
outside multitenancy, or any unit of work after a leaving that did not complete, breaks no rule of entering.
Refusing a unit of work, such as answering a request with an error, and reporting it, run in the same process. Under
[ADR-0005](0005-mechanisms.md), Sprout, not the application, enforces isolation, and refuses an operation whose
isolation it cannot establish.

## Decision

We will have Sprout start and finish each type of [unit of work](../GLOSSARY.md#unit-of-work) it supports, such as a
request, a queued job, a console command or a test, and, when one finishes, [leave](../GLOSSARY.md#leaving) each
[tenant](../GLOSSARY.md#tenant) [entered](../GLOSSARY.md#entering) during it that is still active, in the reverse of
the order they were entered. When a unit of work starts with [stale state](../GLOSSARY.md#stale-state), Sprout will
leave each tenant still active and put back everything else it can, and throw, so that the unit of work does not
run. The process that found the stale state will run no further units of work, and Sprout will report both the stale
state and the stop.

## Alternatives

**Resetting and carrying on.** Sprout would reset, then run the [unit of work](../GLOSSARY.md#unit-of-work), with or
without a log entry or an event to report it. We rejected this because a reset after a failed cleanup cannot be
trusted to be complete, so the unit of work could still run with some of another [tenant](../GLOSSARY.md#tenant)'s
state.

**Refusing only the unit of work that found it.** Sprout would reset and throw, and the process would go on to run
further units of work. We rejected this because those units of work would run after the same reset, which cannot be
trusted to be complete.

**Refusing every unit of work in the process.** The process would keep running, and refuse each unit of work while
the [stale state](../GLOSSARY.md#stale-state) could not be confirmed gone. We rejected this because nothing can
confirm it, so the process would refuse everything while appearing to run.

**Rebuilding instead of resetting.** Sprout would discard everything it built for tenants, rebuild it from the
application's untouched state, and carry on. We rejected this because Sprout cannot rebuild state held by other
code, such as in static properties, so the rebuild could not be trusted to be complete either.

## Consequences

Easier:

- State that a failed cleanup did not put back will not reach a later [unit of work](../GLOSSARY.md#unit-of-work) in
  the same process.

Harder:

- Each failed cleanup will cost the unit of work after it, even one that has nothing to do with multitenancy, such
  as a health check, and the process that found the [stale state](../GLOSSARY.md#stale-state) will stop taking work.
- In a test run, one failed cleanup will stop the rest of the run in that process.
- A type of unit of work that Sprout does not start and finish, such as one an application adds, will not be checked
  when it starts, and a [tenant](../GLOSSARY.md#tenant) still active when it finishes will be stale state for the
  next unit of work Sprout starts in that process, which will stop the process.

Constrained:

- Every tenant [entered](../GLOSSARY.md#entering) during a unit of work that Sprout finishes, and still active when
  it finishes, will be [left](../GLOSSARY.md#leaving) then.
- Sprout will not run a unit of work that starts with stale state, or any later one in the same process.
- Sprout will report each time it finds stale state, and each process it stops because of it.
