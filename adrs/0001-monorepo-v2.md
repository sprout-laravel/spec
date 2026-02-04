# ADR-0001: Monorepo v2

- **Status:** Accepted
- **Date:** 2026-02-04
- **Supersedes:** N/A
- **Superseded by:** N/A

## Context

The original plan was to develop the series of Sprout addons as separate packages that could be installed individually
dependent on the requirements of the user. It was likely that the majority of people would require at least one of the
addons, so it included an additional step to get started.

There was also an outstanding deprecation within the codebase that had been flagged quite early and was waiting for
full deprecation in the next major release.

## Decision

The decision was made to develop all of the addons within a single monorepo codebase that functions as a single package,
providing all functionality and reducing the number of steps required to get started.

## Consequences

Due to the massive restructure of the codebase, which affected the namespaces, the public API was broken, so a bump to
the major version was required.

### Positive

- Lower barriers to entry for new users
- Easier to develop and maintain

### Negative

- Required a major version bump
- They've been referred to as "addons" everywhere

### Neutral

- None

## Alternatives Considered

The decision was between this approach and sticking with the original plan.

### Alternative 1

The other alternative was to develop the addons as separate packages and keep them separated only to pull in when
needed.

This was rejected for the following reasons:

- It would require a lot of maintenance overhead
- The development process would be more complex
- It made it more complicated for users to get started
- Personal preference, I'd honestly decided against the idea

## References

- The issue for the change: sprout-laravel/sprout#120
