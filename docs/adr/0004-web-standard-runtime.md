# 4. Web-standard core runtime

**Status:** Accepted, 2026-11-22

## Context

Bun is the build and test tool, but the first draft never said where the library itself runs. Developers deploy to Node, Bun, Deno, Cloudflare Workers and Vercel Edge.

## Decision

Core uses only web-standard APIs (`Request`, `Response`, `fetch`, Web Crypto, streams) and is tested on Node 20+, Bun, Deno and Cloudflare Workers in CI. `chat.handler` is a plain `Request => Response`, with small helpers for Next.js, Node/Express and per-runtime WebSocket upgrades.

## Consequences

- No Node built-ins in core. Adapters that can't run on the edge must say so and fail at startup.
- CI needs a runtime matrix, and tests can't depend on Bun-only APIs (ADR 13).
