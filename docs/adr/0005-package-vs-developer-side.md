# 5. Package side vs developer side

**Status:** Accepted, 2026-11-22

## Context

The package is meant to be "Better Auth for chat": it generates schema, API and types, and the developer runs it. The first draft also planned first-party transports for Redis, Ably, Pusher, Durable Objects and Supabase, and several adapters per vendor area (push, GIFs, search). That is infrastructure work, not package work.

## Decision

The package owns schema, endpoints, types, event contracts and provider interfaces. Whatever runs outside the developer's server is the developer's side.

- **Realtime:** the package defines the event contract (typed events, `seq`, gap-fill) and serves SSE and WebSocket from its own handler. Fan-out across servers and serverless delivery go through `defineTransport`; Redis, Ably and others are docs recipes or community packages.
- **Vendor areas:** each area (file storage, GIFs, push, search, calls, AI models) gets a `define*Provider` interface plus exactly one first-party reference adapter, so it works out of the box.

## Consequences

- Fewer packages to maintain, and each interface has a contract suite third parties can run.
- Serverless deployments need the developer to pick a transport; the docs must make that obvious.
