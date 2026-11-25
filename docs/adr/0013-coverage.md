# 13. Coverage: 100% including branches, documented ignores

**Status:** Accepted, 2026-11-22

## Context

The first draft enforced 100% lines and functions with Bun, which doesn't measure branches. A hard line gate with no branch coverage rewards tests that touch code over tests that check behaviour.

## Decision

- Core, adapters and first-party plugins must reach 100% lines, functions and branches.
- CI adds a second, branch-measuring coverage pass. Tests import test APIs through a local shim, so the same files run under Bun and the second runner.
- Coverage ignores need a one-line reason and are allowed only for generated files and provably unreachable defensive branches; CI reports new ones on each PR.
- Contract and integration suites are the real gate.

## Consequences

- Two test runners and slower CI.
- The shim also serves the runtime matrix (ADR 4), since tests can't rely on `bun:test` directly.
- The second runner is still to be picked (spec open items).
