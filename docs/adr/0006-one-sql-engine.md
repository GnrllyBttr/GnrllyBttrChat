# 6. One Kysely-based SQL engine

**Status:** Accepted, 2026-11-22

## Context

The spec offers Kysely, Drizzle and Prisma adapters across Postgres, MySQL and SQLite. Writing the chat layer (atomic `seq`, one DM per pair, batched unread counts) three times would triple the code and the contract-suite runs.

## Decision

One Kysely-based engine implements the generic and chat layers for all three dialects. The Drizzle and Prisma adapters only hand their connection to that engine.

## Consequences

- One chat-layer implementation per dialect, tested once.
- The Drizzle and Prisma adapters depend on reaching the underlying driver connection; if either ORM hides it, that adapter needs a workaround.
