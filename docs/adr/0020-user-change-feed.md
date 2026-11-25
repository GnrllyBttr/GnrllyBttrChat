# 20. Per-user change feed

**Status:** Accepted, 2026-11-25

## Context

Per-conversation `seq` lets a client catch up on messages, but a user in 200 conversations would need 200 calls on reconnect, and could miss conversation-level changes: being added, removed, a rename.

## Decision

Core keeps a `userFeed` table with a gap-free `userSeq` per user. Conversation-level changes (added, removed, updated, unread changed, new last message) go in the feed. `GET /sync?since=<userSeq>` returns them in one call; messages are then fetched only for open conversations. Feed entries older than `sync.retention` (default 30 days) may be pruned; an older cursor gets `SYNC_RESET`.

## Consequences

- One extra write per affected participant per conversation-level change. In big conversations a new message would fan out to every member, so "new last message" entries are coalesced: at most one unread entry per user per conversation between syncs.
- The scale targets (spec §11) must include this fan-out.
