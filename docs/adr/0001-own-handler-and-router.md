# 1. Own handler and router, not Better Auth core

**Status:** Accepted, 2026-11-22

## Context

The design borrows Better Auth's shape (config-driven schema, plugins, adapters, typed client), but chat has its own needs: long-lived SSE and WebSocket streams, gap-fill, and versioning independent of Better Auth. Better Auth's endpoints run on better-call, a standalone router; Hono was another option.

## Decision

GnrllyBttr Chat has its own handler, adapters, CLI and a small router with typed endpoints, middleware and client inference. It does not build on Better Auth's core, better-call or Hono. Better Auth stays a first-class, optional integration (`chat-better-auth`).

## Consequences

- Full control over streaming, routing and inference, with no upstream router to track.
- More code to own and keep at 100% coverage; the router needs its own test suite.
- Plugin authors still get a Better Auth-like shape (`definePlugin`, `createChatEndpoint`), but not identical internals.
