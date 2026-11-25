# 2. Audience: any app with chat

**Status:** Accepted, 2026-11-22

## Context

The first draft never said who the package is for, which left no basis for scoping. Candidates were SaaS in-app messaging, team chat, AI-first apps and consumer messengers.

## Decision

SaaS in-app messaging is the lead use case, but the package must serve any app with chat: simple 1:1 messengers, social group chats, and Slack, Discord or Teams-style team chat. SaaS-specific features (such as `tenantId`) are optional, so a simple app never sees them.

## Consequences

- Messaging targets parity with Slack, Discord, Teams and WhatsApp (ADR 3).
- A feature must be justified by at least one of the four app types in spec §1.
