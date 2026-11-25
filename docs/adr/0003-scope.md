# 3. 1.0 includes everything, with a core/plugin split

**Status:** Accepted, 2026-11-22. The core/plugin split is superseded by [ADR 17](0017-minimal-core.md); the 1.0 scope stands.

## Context

A lean 1.0 (core messaging only, big plugins in 1.x) was considered, to reach a stable API sooner for a solo maintainer. The alternative was to ship the full feature set before tagging 1.0.

## Decision

1.0 ships every feature in the spec: messaging parity, media, AI, calls, E2EE (experimental, ADR 11) and the apps protocol (ADR 12). Heavy or vendor-tied features stay opt-in plugins or separate packages; messaging parity (threads, channels, mentions, forwarding) is core.

## Consequences

- A long 0.x period before 1.0. The plugin API must stabilise early, since everything else builds on it.
- Apps install only the plugins they use, so vendor SDKs never become core dependencies.
