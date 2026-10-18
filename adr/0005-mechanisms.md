---
id: ADR-0005
title: Provide mechanisms, not policy
status: accepted
created: 2026-10-18
decided: 2026-10-18
depends: []
updates: []
obsoletes: []
---

# ADR-0005: Provide mechanisms, not policy

## Context

Applications that serve many [tenant](../GLOSSARY.md#tenant)s differ in what a tenant is and how one is managed. A
tenant can be, for example, an organisation, a store or a team, stored as an Eloquent model, a table read through
the query builder, or a record in another service. One application creates a tenant's database when it signs up,
another in a queued job once it has paid, another by hand. Whether a tenant may be served at a given moment, such as
one that is suspended or has not paid, is a business rule that differs between applications.

Isolation does not differ in the same way. Whether one tenant's data or state, such as its files, cache entries or
session, can reach another depends on how the boundary between them is applied, not on the application's business. A
boundary that the application has to remember to apply in every place leaks wherever it is forgotten.

## Decision

We will provide the mechanisms an application needs to serve many [tenant](../GLOSSARY.md#tenant)s, and leave policy
and process to the application. Sprout decides how isolation is enforced, such as making a tenant the
[active tenant](../GLOSSARY.md#active-tenant), keeping its state apart from every other tenant's, and refusing an
operation whose isolation it cannot establish. The application decides the rest, such as what a tenant is and where
it is stored, which tenants may be served and how a refusal is answered, which values a tenant may configure, and
when and in what order a tenant's resources are created. Where Sprout needs an answer to one of these questions, it
asks through a contract, an event or an option, and supplies at most a default that the application can replace.

## Alternatives

**A package that manages the whole life of a tenant.** Sprout would ship a [tenant](../GLOSSARY.md#tenant) model to
extend, a sign-up flow, and a provisioning pipeline that runs when a tenant is created. We rejected this because
these choices vary too much between applications, and an application whose process differs would have to work around
the package.

**Building blocks without enforcement.** Sprout would provide the pieces, such as finding the tenant for a request
and scoping queries, and leave the application to apply them. We rejected this because isolation is what the package
exists to provide: if applying it is optional, an application leaks data through any omission, and Sprout cannot
detect it.

## Consequences

Easier:

- An application will be able to keep its own [tenant](../GLOSSARY.md#tenant) model, schema and storage, and its own
  sign-up and provisioning process, without working around Sprout.
- A feature request will have a test: if it asks Sprout to make a policy decision, it belongs to the application or to
  a separate tool built on Sprout.

Harder:

- Applications will have to write their own policy, such as the rule that refuses a suspended tenant, and their own
  process, such as provisioning a new one.
- Every point where the application decides will need a contract, an event or an option, and each contract is an
  [extension point](../GLOSSARY.md#extension-point) kept stable under [ADR-0003](0003-extension-points.md).

Constrained:

- Sprout will not require a particular model, schema or storage for tenants.
- Sprout will not decide which tenants are served, or when anything is provisioned.
- Isolation will be enforced by Sprout, not left to the application to apply.
