# 25. Lockstep versions during 0.x

**Status:** Accepted, 2026-11-25

## Context

About 15 first-party packages will move fast during 0.x. Independent versions would need a compatibility matrix.

## Decision

All first-party packages share one version (a Changesets `fixed` group) until 1.0. During 0.x a minor release may break things, with a migration note each time; patches never break. Independent versions start at 1.0.

## Consequences

- "Use the same version everywhere" is the whole compatibility story.
- Packages with no changes still get version bumps.
