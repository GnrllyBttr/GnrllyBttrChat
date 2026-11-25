# 17. Minimal core: engine plus 1:1 text chat

**Status:** Accepted, 2026-11-25. Supersedes the core/plugin split in ADR 3.

## Context

ADR 3 put messaging parity in core: groups, channels, threads, replies, reactions, mentions and forwarding. The goal is a base package that stays simple, like Better Auth's core, with features added by plugins.

## Decision

The base package `@gnrllybttr/chat` is the engine plus 1:1 text chat:

- router, handler, `chat.api`, plugin API
- schema, CLI, SQL engine and adapters
- auth resolvers, tenancy
- DMs, text messages with spans, send/edit/delete, `clientMessageId`
- read state, unread counts, `seq`, the user feed, SSE and WebSocket
- user delete and export, import
- rate limits, Origin checks, logging

Everything else is a plugin, including group chat, replies, reactions, mentions, system messages, bot identity, mute and notification preferences, conversation management, delete-for-me, the audit log, AI, calls, E2EE and apps. All first-party plugins still ship before 1.0 (ADR 3's scope stands).

## Consequences

- Plugins must be able to add columns to core tables (for example `message.replyToId`) and register conversation types, message kinds, span types and events.
- `definePlugin` needs `dependsOn`, checked at startup (`chat-ai` needs `sender-identity`, `channels` needs `groups`).
- A one-line `createChat({ database, auth })` gives a working 1:1 chat; every other feature is one import away.
