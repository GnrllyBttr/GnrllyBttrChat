# 19. Idempotent send with `clientMessageId`

**Status:** Accepted, 2026-11-25

## Context

On a flaky network a client retries a send and creates a duplicate. Mobile apps and optimistic UI both need a safe retry.

## Decision

Every send carries a `clientMessageId`, unique per sender per conversation and enforced by a unique index. A send with an id that's already saved returns the saved message. `chat.api` sends take it too.

## Consequences

- The client renders the message at once and swaps in the saved one.
- The contract suite tests concurrent duplicate sends.
- A generic `Idempotency-Key` for every endpoint was rejected as more storage for less benefit.
