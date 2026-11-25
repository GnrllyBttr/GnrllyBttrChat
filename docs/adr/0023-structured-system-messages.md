# 23. Structured system messages

**Status:** Accepted, 2026-11-25

## Context

System messages ("Sam was added", "Order shipped", "Call ended, 4 min") stored as English text can't be translated.

## Decision

The `system-messages` plugin stores `{ type, params }` plus a fallback `text`. Clients translate and render by `type`; push and search use the fallback. `chat.api` may still post free text for app-specific notices.

## Consequences

- Every plugin that posts system messages documents its types and params.
- Clients without a renderer for a type show the fallback text.
