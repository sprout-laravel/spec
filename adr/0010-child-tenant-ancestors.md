---
id: ADR-0010
title: Enter a child tenant only beneath its own ancestors
status: accepted
created: 2026-10-18
decided: 2026-10-19
depends: [ADR-0001, ADR-0005, ADR-0006, ADR-0007]
updates: [ADR-0001, ADR-0007]
obsoletes: []
---

# ADR-0010: Enter a child tenant only beneath its own ancestors

## Context

A [child tenant](../GLOSSARY.md#child-tenant) belongs to its parent, and a parent can itself be a child, so a
[tenant](../GLOSSARY.md#tenant) can have a chain of [ancestor](../GLOSSARY.md#ancestor)s up to one with no parent.
Nesting happens between different [tenancies](../GLOSSARY.md#tenancy), such as an organisation and its stores, and
within one tenancy, such as a reseller and its customers. A tenant that can have a parent says which tenant is its
parent. A child's state can sit within its ancestors', such as a child's tables in its parent's own database, and
[leaving](../GLOSSARY.md#leaving) a tenant puts back the state from before it was entered.

Under [ADR-0001](0001-tenant-switching.md), code can [enter](../GLOSSARY.md#entering) a tenant only when its tenancy
has no [active tenant](../GLOSSARY.md#active-tenant), or enter a child beneath the active parent, and under
[ADR-0006](0006-entering.md), checking a child's parent belongs to entering. [ADR-0001](0001-tenant-switching.md)
alone lets a child be entered whenever its own tenancy has no active tenant, whatever is active above it, so one
organisation's store could be entered while another organisation is active. A child can be reached with its parent
active, such as a store [identified](../GLOSSARY.md#identification) from a request on its organisation's domain, or
with none of its ancestors entered, such as a queued job for a store run by a worker in which nothing is active. A
child reached through a value from outside, such as a route parameter, can belong to a different parent from the one
that is active, such as one organisation's store reached on another organisation's domain through a tampered
parameter.

Under [ADR-0007](0007-refusal.md), the application can refuse a tenant as it is entered, a refused tenant does not
become active, and [unrefusable entering](../GLOSSARY.md#unrefusable-entering) is explicit in the code that uses it.

## Decision

We will check, whenever a [child tenant](../GLOSSARY.md#child-tenant) is [entered](../GLOSSARY.md#entering), and
once each of its [ancestor](../GLOSSARY.md#ancestor)s has been entered, that it is entered beneath its parent, and
throw otherwise, with no way to turn the check off. When a child is entered with ancestors that are not already
entered, Sprout will follow each [tenant](../GLOSSARY.md#tenant)'s parent to the root, enter each of those ancestors
top down before the child, and [leave](../GLOSSARY.md#leaving) them with it, in reverse order. Sprout will only fill
gaps: a [tenancy](../GLOSSARY.md#tenancy) along the chain whose [active tenant](../GLOSSARY.md#active-tenant) is not
one of the child's ancestors is a mismatch and throws, and an ancestor entered before the child is not left with it.
If the child's entering fails, Sprout will leave every ancestor it entered for it. Each ancestor's entering can be
refused as any other can, and refusing an ancestor refuses the child. When the child's entering is
[unrefusable entering](../GLOSSARY.md#unrefusable-entering), so is each ancestor's entering for it. Leaving a tenant
while a tenant entered after it is still entered will throw, so a parent is never left while a child is entered
beneath it.

## Alternatives

**Trusting the application to enter the right parent.** Sprout would enter a
[child tenant](../GLOSSARY.md#child-tenant) beneath whatever [tenant](../GLOSSARY.md#tenant) was active, and leave
the application to check the relationship. We rejected this because a check the application missed would let a
request for one parent reach another parent's child, which Sprout can catch on every
[entering](../GLOSSARY.md#entering), and [ADR-0005](0005-mechanisms.md) has Sprout, not the application, enforce
isolation.

**Throwing when the parent is not active.** Entering a child would require its [ancestor](../GLOSSARY.md#ancestor)s
to be entered already, and the caller would enter them first, from the root. We rejected this because every queued
job, command and test that reaches a child would repeat the same steps: loading every ancestor, entering them from
the root, and leaving them in reverse order.

**Keeping an ancestor entered for a child while anything is entered beneath it.** Leaving a child would leave the
ancestors Sprout entered for it only once no other tenant was entered beneath them, so a tenant could be left before
one entered after it. We rejected this because an ancestor kept for another child would stay entered after every
child it was entered for had been left, until the unit of work finished, and putting back a tenant's state while a
tenant entered after it is still entered would also put back that tenant's changes.

## Consequences

Easier:

- [Entering](../GLOSSARY.md#entering) a [child tenant](../GLOSSARY.md#child-tenant) that belongs to a different
  parent from the active one will fail, wherever the child came from.
- Code that reaches a child, such as [restoration](../GLOSSARY.md#restoration) for a queued job, will hand only the
  child to entering, which will find its [ancestor](../GLOSSARY.md#ancestor)s from each
  [tenant](../GLOSSARY.md#tenant)'s parent, so a queued job will need to carry only the child.
- A refusal rule for a parent, such as one that refuses a suspended organisation, will also apply to a child whose
  parent Sprout enters for it.

Harder:

- Entering a child with ancestors that are not already entered will look up and enter each of them.
- A child whose ancestor is refused will be reachable only through
  [unrefusable entering](../GLOSSARY.md#unrefusable-entering), of the child or of that ancestor.
- Code that enters and leaves several tenants itself will have to [leave](../GLOSSARY.md#leaving) them in the
  reverse of the order it entered them.

Constrained:

- No child will be entered while a tenant that is not one of its ancestors is the
  [active tenant](../GLOSSARY.md#active-tenant) of an ancestor's [tenancy](../GLOSSARY.md#tenancy).
- Entering a child will never replace an active tenant to make way for one of its ancestors.
- Leaving a tenant while a tenant entered after it is still entered will throw.
