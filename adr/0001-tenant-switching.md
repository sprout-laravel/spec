---
id: ADR-0001
title: Never replace the active tenant
status: proposed
created: 2026-10-03
decided:
depends: []
updates: []
obsoletes: []
---

# ADR-0001: Never replace the active tenant

## Context

Within each [unit of work](../GLOSSARY.md#unit-of-work), Sprout sets up everything that depends on the
[tenant](../GLOSSARY.md#tenant) for the [active tenant](../GLOSSARY.md#active-tenant) of each
[tenancy](../GLOSSARY.md#tenancy), such as which database records can be seen, where files are stored and which
cache is used.

Some of what is set up belongs to the request itself and cannot be handed to another tenant. The session, the
cookies and the authenticated user all belong to the tenant the request arrived through, and a login is tied to the
tenant that issued it. Other things, such as database queries, the cache and stored files, can be pointed at a
different tenant.

There are situations where it can look as though one tenant needs to replace another partway through a unit of work,
such as:

- an administrator looking at a particular tenant's data
- a webhook arriving at an endpoint shared by every tenant, where the tenant is only known once the payload has been
  read
- a reseller looking at the data of the customers it resells to, each of which is a
  [child tenant](../GLOSSARY.md#child-tenant) of the reseller
- one tenant acting on another, such as sharing a document or notifying another tenant's user
- an administrator impersonating one of a tenant's users
- tests that check one tenant cannot see another tenant's data

None of these needs an active tenant to be replaced. The administrator and the webhook start with no tenant, so they
can enter one. The reseller is the parent of its customers, so it can enter a customer beneath itself. Work for
another tenant can be handed to that tenant as a separate queued job, which runs as its own unit of work.
Impersonation is a new request on the tenant's own domain, and tests can enter and leave tenants one after another.

## Decision

We will never replace a [tenant](../GLOSSARY.md#tenant) that is active for a [tenancy](../GLOSSARY.md#tenancy)
before its [unit of work](../GLOSSARY.md#unit-of-work) has finished, whatever type of unit of work it is. Code will
be able to enter a tenant only when its tenancy has no [active tenant](../GLOSSARY.md#active-tenant), or enter a
[child tenant](../GLOSSARY.md#child-tenant) beneath the active parent, and leaving a tenant will always put things
back as they were, even if something goes wrong along the way. When something needs to be done for a tenant other
than the active one, we will hand it to that tenant as a separate queued job.

## Alternatives

**Replacing the tenant, and moving whatever can be moved.** The database, cache and files would switch to the new
[tenant](../GLOSSARY.md#tenant), while the session, cookies and logged-in user stayed with the original. We rejected
this because the [unit of work](../GLOSSARY.md#unit-of-work) would then be working for two tenants at the same time,
and anything that failed to fully let go of the first tenant would expose its data to the second. None of the
situations above needs it.

**Allowing a replacement only when nothing tied to the request is in use.** This is safer, but everything it would
allow is already possible by entering a tenant when none is active, or by handing the work to the other tenant as a
job. We rejected it because it adds a rule without making anything new possible.

**Applying the rule to requests only.** A queued job or command that needs to act for another tenant has the same
answer as a request: hand the work to that tenant. We rejected this because one rule for every type of unit of work
is easier to understand, test and explain than one with exceptions.

## Consequences

Easier:

- Everything in Sprout that depends on the [tenant](../GLOSSARY.md#tenant) will only need to handle a tenant being
  entered or left, never one being swapped for another.
- Keeping tenants apart will rest on each [unit of work](../GLOSSARY.md#unit-of-work) starting and finishing
  cleanly, which Sprout already has to get right for applications that keep running between requests, such as those
  using Octane, and for queue workers.

Harder:

- Anything that acts across tenants will go through the queue, so a request will not see the result of work done for
  another tenant before it finishes.
- Tools that show several tenants' data, such as an administration dashboard, will have to enter and leave each
  tenant in turn, starting from a point where no tenant is active.

Constrained:

- Anything that sets the [active tenant](../GLOSSARY.md#active-tenant) will refuse to do so while a tenant is
  already active for that [tenancy](../GLOSSARY.md#tenancy), unless it is entering a
  [child tenant](../GLOSSARY.md#child-tenant) beneath it.
