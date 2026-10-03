---
id: ADR-0003
title: Keep extension points stable within a major version
status: accepted
created: 2026-10-03
decided: 2026-10-03
depends: []
updates: []
obsoletes: []
---

# ADR-0003: Keep extension points stable within a major version

## Context

Applications and third-party packages extend Sprout. They add new ways of identifying a
[tenant](../GLOSSARY.md#tenant), loading tenants, separating tenant data, making services tenant-aware, providing
and storing [per-tenant configuration](../GLOSSARY.md#per-tenant-configuration), and provisioning resources for
tenants. Sprout's own built-in parts register through the same interfaces and methods, and other projects, including
a developer toolkit and a provisioning orchestration tool, are planned on top of them.

When a package implements one of Sprout's interfaces, it breaks as soon as that interface gains a new method, even
if nothing else it relies on has changed. Without a clear line between what is meant to be built on and what is not,
a package author cannot tell which parts of Sprout they can safely depend on.

## Decision

We will mark every [extension point](../GLOSSARY.md#extension-point) in Sprout as one, and will not break any of
them outside a major version. An extension point interface will not gain new methods in a minor release, and new
capabilities will arrive as new interfaces. Everything else is internal, and can change in any release.

## Alternatives

**Treating every public class and method as stable.** We rejected this because every change to Sprout's internals
would become a breaking change, so the internals could only change in a major release.

**Making no formal promise.** We rejected this because third-party packages could break in any release, and
supporting third-party packages means giving them something they can rely on.

## Consequences

Easier:

- Package authors will know which parts of Sprout they can depend on.
- Sprout's internals will be free to change in minor releases.

Harder:

- Adding a capability to an existing [extension point](../GLOSSARY.md#extension-point) will mean introducing a new
  interface, rather than adding a method.
- Each design will need to decide which of its parts are extension points.

Constrained:

- An extension point interface will not gain methods outside a major version.
- Sprout's built-in parts will register through extension points, the same way third-party packages do.
