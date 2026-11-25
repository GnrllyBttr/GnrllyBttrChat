# 21. RPC-style HTTP API

**Status:** Accepted, 2026-11-25

## Context

The API surface was undefined. The options were REST resources (`POST /conversations/:id/messages`) or Better Auth-style action paths.

## Decision

Action paths that map one-to-one onto `chat.api`: `chat.api.messages.send` is `POST /messages/send`. `GET` for reads, `POST` for writes. Errors are `{ error: { code, message, details? } }` with a fixed code list. Default limits are configurable.

## Consequences

- Plugin actions (forward, vote, accept a request) fit naturally.
- Client types infer from one endpoint map.
- Less familiar to REST-first developers; the docs list every endpoint.
