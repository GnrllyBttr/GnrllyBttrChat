# 16. Message requests as a plugin

**Status:** Accepted, 2026-11-22

## Context

Social and 1:1 apps need Instagram or WhatsApp-style message requests: a DM from a stranger waits until the recipient accepts. SaaS apps usually don't.

## Decision

A `message-requests` plugin adds a `canStartDirect` hook (allow, request, deny), its own `messageRequest` table and its endpoints. On accept, held messages move into the real DM in order. Core is unchanged.

## Consequences

- Core's DM path must leave a seam for the plugin to intercept `createDirectOnce` and the first send.
