---
id: ADR-0004
title: Upgrade from v1 with a guide and Rector rules
status: accepted
created: 2026-10-03
decided: 2026-10-03
depends: []
updates: []
obsoletes: []
---

# ADR-0004: Upgrade from v1 with a guide and Rector rules

## Context

Version 2 of Sprout is being rebuilt from scratch. Breaking changes from version 1 are allowed, as long as there is
a way for applications using version 1 to upgrade. Some of the changes are mechanical, such as namespaces, renamed
methods and the shape of configuration, while others change how an application uses Sprout.

## Decision

We will provide an upgrade guide for moving from version 1 to version 2, along with Rector rules that make the
mechanical changes automatically. We will not provide a compatibility layer for running version 1 code on version 2.

## Alternatives

**An upgrade guide on its own.** We rejected this because every application would have to make the same mechanical
changes by hand.

**A compatibility layer that lets version 1 code run on version 2.** We rejected this because it would keep version
1's design alive inside a version that is being rebuilt from scratch.

## Consequences

Easier:

- Version 2 will carry nothing whose only purpose is compatibility with version 1.
- The mechanical part of upgrading will be automated.

Harder:

- The Rector rules will need writing, and testing against real version 1 applications.
- Changes that Rector cannot make will rely on the guide alone.

Constrained:

- Version 2 will contain no code whose only purpose is to accept the way version 1 was used.
