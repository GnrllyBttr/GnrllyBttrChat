# 14. Naming

**Status:** Accepted, 2026-11-22

## Context

"GnrllyBttrChat" is hard to say, spell and search for. The npm scope `@gnrllybttr/chat` reads fine.

## Decision

The product is "GnrllyBttr Chat"; docs say it once, then "Chat". The package is `@gnrllybttr/chat`, the config function is `createChat()` and the client is `createChatClient()`. `GnrllyBttr` stays the org brand.

## Consequences

- Code examples read naturally, and the brand lives in the package scope.
- The plugin npm keyword is `gnrllybttr-chat-plugin`.
