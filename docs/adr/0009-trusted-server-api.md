# 9. Trusted server API (`chat.api`)

**Status:** Accepted, 2026-11-22

## Context

Backends need to act without a user session: system notices ("Order shipped"), welcome messages, imports, chats created from webhooks.

## Decision

Every HTTP endpoint has a matching `chat.api` function. Calls act `as` a user id, skip the `can*` permission hooks, but still run `before*` and `after*` hooks, validation and publishing.

Amended by ADR 17, 23 and 24: acting `as: "system"` and the `system` message kind come from the `sender-identity` and `system-messages` plugins, and the audit trail from the `audit-log` plugin.

## Consequences

- One code path for HTTP and server calls, with the same types.
- `chat.api` must never be reachable through the HTTP handler; tests must prove it.
