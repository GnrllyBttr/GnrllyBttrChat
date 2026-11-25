# 24. Sender identity plugin for bots and system

**Status:** Accepted, 2026-11-25

## Context

AI assistants, app bots and the `system` sender need names and avatars, but core has no user table: profiles come from `resolveUsers`, which only knows human users.

## Decision

The `sender-identity` plugin adds `message.senderType` (`user`, `bot`, `system`) and a `chatBot` table (id, tenantId, name, avatarUrl, source, sourceId). Profile lookups check bots first, then `resolveUsers`. `system-messages`, `chat-ai` and `chat-apps` depend on it.

## Consequences

- Core stays human-only; apps without bots never see the table.
- One shared bot model, so the client needs a single profile lookup.
