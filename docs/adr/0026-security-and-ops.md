# 26. Security and operations baseline

**Status:** Accepted, 2026-11-25

## Context

The first spec missed several things: `EventSource` can't send an `Authorization` header; cookie auth needs CSRF protection; in-memory rate limits don't work on serverless or across servers; the audit trail had no home; there were no scale targets or observability; and moderators can't read encrypted messages.

## Decision

- **Stream auth:** cookie auth as-is. Bearer tokens use a fetch-based SSE reader, or `POST /stream/ticket` for a single-use, 30-second ticket.
- **CORS and CSRF:** a `trustedOrigins` list drives CORS headers and an `Origin` check on every cookie-authenticated mutation.
- **Rate limits:** `rateLimit.storage` is memory (default), database, or `defineRateLimitStore`.
- **Audit:** the `audit-log` plugin records `chat.api` and app actions.
- **Observability:** a core `logger` option that never logs bodies; the `opentelemetry` plugin adds spans.
- **Scale:** written targets (10k members per conversation, 100 messages/s per conversation, unread counts across 500 conversations in one query, debounced read-state writes), verified by a load test before 1.0.
- **Encrypted reports:** the reporter's client attaches the decrypted message.

## Consequences

- The default memory rate-limit store is unsafe on serverless; docs and a startup warning on known serverless runtimes must say so.
- The load test is a 1.0 gate.
