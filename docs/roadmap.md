# Roadmap

How the [spec](../spec.md) ships, release by release. Every first-party package shares one version until 1.0 ([ADR 25](adr/0025-versioning.md)). During 0.x a minor release may break things; each one ships a migration note covering API and schema changes.

How releases are ordered:

- Each minor release is usable on its own and demoable.
- Anything every row or plugin depends on ships first: tenancy, the body format, sync cursors, user delete and export.
- NoSQL adapters come after the messaging plugins, so each adapter is written once against the full feature set.
- AI comes before calls because it reuses what's already built (bots, streaming events). E2EE comes last because it touches every other feature.

| Version      | Theme          | Ships |
| ------------ | -------------- | ----- |
| **0.1.0**    | Engine + 1:1   | Repo foundation and CI (runtime matrix, coverage gate, release pipeline). Router, `chat.handler`, Next.js and Node helpers. Kysely engine (Postgres, SQLite) with the Drizzle adapter; memory adapter. Schema system, `generate`, `migrate`. JWT/JWKS and custom resolvers. Tenancy. DMs; text with spans; send, edit, delete; `clientMessageId`. Read state and unread counts. `seq`, gap-fill, `userFeed` and `/sync`. SSE and stream tickets. `chat.api`, user delete and export. Plugin API. Rate-limit store, `trustedOrigins`, `logger`. Framework-agnostic client. `chat-testing` with the storage contract suite. |
| **0.2.0**    | Real apps      | `chat-better-auth`. MySQL. Kysely and Prisma adapters. WebSocket and upgrade helpers for Node, Bun, Deno, Workers. `defineTransport` and its contract suite. React and React Native bindings. Example: SaaS buyer-seller (Next.js, Better Auth, Postgres). |
| **0.3.0**    | Group chat     | Plugins: `groups` (roles, conversation management), `replies`, `reactions`, `mentions` (with `@channel` and badges), `presence` (receipts toggle), `notifications`, `message-controls`, `mark-unread`, `sender-identity`, `system-messages`, `audit-log`. Example: social group chat (SvelteKit). |
| **0.4.0**    | Team chat      | Plugins: `channels`, `threads`, `mention-inbox`, `role-mentions` and `@here`, `forwarding`, `links`, `share-links`, `invites`, `message-requests`, `moderation`. |
| **0.5.0**    | Media          | `chat-media` (S3-compatible). Plugins: `gifs` (Klipy), `link-previews`, `custom-emoji`, `pins`, `polls`, `scheduled-messages`. Example: 1:1 messenger (Expo). |
| **0.6.0**    | Reach          | Plugins: `push` (Web Push), `search` (database full-text), `disappearing-messages`, `opentelemetry`. `chat.api.import`. Vue, Svelte and Solid bindings. `chat-ui`. Example: team chat (React, Hono). |
| **0.7.0**    | More databases | `chat-mongodb`, `chat-dynamodb`, `chat-firestore`, each passing the full contract suite. |
| **0.8.0**    | AI             | `chat-ai`: assistants, streaming, tools, MCP, usage limits, fake model. |
| **0.9.0**    | Calls          | `chat-calls`: 1:1, group, screen share, recording (LiveKit). |
| **0.10.0**   | Apps           | `chat-apps`, `chat-app-sdk`: manifests, installs, scopes, signed delivery, blocks, slash commands. |
| **0.11.0**   | Encryption     | `chat-e2ee` (experimental): MLS groups, multi-device, encrypted forwarding, encrypted-chat reports, encrypted calls. |
| **1.0.0-rc** | Freeze         | Docs site, all four example apps green, plugin starter (`plugin init`), load test at the scale targets, API freeze. Release candidates until no breaking change is needed. |
| **1.0.0**    | Stable         | Independent package versions start here. |

## Before 0.1.0

- Create the `@gnrllybttr` npm org.
- Pick the branch-coverage runner and prove the test shim.

## After 1.0

- App registry: repo, review process, publisher verification, delisting.
- `chat-e2ee` stable, after an independent security review.
- Native VoIP push for ringing locked phones.
