# 15. No built-in conversation subject

**Status:** Accepted, 2026-11-22

## Context

SaaS chats are usually about a record: an order, a booking. A core `subject: { type, id }` field with an idempotent `findOrCreate({ subject })` was proposed.

## Decision

No built-in subject. Developers link conversations to their records with `additionalFields`, which must support `index: true` and `unique: true` so the lookup is fast and can be made idempotent.

## Consequences

- Core stays smaller and neutral across app types.
- The SaaS docs need a recipe for "one chat per order" using a unique additional field.
