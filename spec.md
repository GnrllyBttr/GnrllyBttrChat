# GnrllyBttr Chat: 1.0 Spec

Requirements for GnrllyBttr Chat 1.0 (`@gnrllybttr/chat`). The why is in [README.md](README.md), the release plan in [docs/roadmap.md](docs/roadmap.md), sourced findings in [docs/research/](docs/research/), and each decision's reasoning in [docs/adr/](docs/adr/).

Requirement words: **must** is required, **should** is expected unless an ADR says otherwise, **won't** is out of scope for 1.0.

## 1. What it is

A TypeScript chat backend that runs inside the developer's own server and stores everything in their own database: Better Auth, but for chat. One config produces the database schema, the HTTP and realtime API, the server and client types, and the runtime validation.

**Who it is for:** any app with chat in it ([ADR 2](docs/adr/0002-audience.md)):

| App                   | Example                                     | Plugins it leans on                                   |
| --------------------- | ------------------------------------------- | ----------------------------------------------------- |
| SaaS in-app messaging | Buyer and seller, coach and client          | Tenancy (core), system messages, moderation, media    |
| 1:1 messenger         | A simple private chat app                   | Presence, message requests, media, E2EE               |
| Social group chat     | Group chat inside a social or community app | Groups, mentions, reactions, media, GIFs              |
| Team chat             | Slack, Discord or Teams style               | Groups, channels, threads, mention inbox, apps        |

**Minimal core** ([ADR 17](docs/adr/0017-minimal-core.md)): the base package is the engine plus 1:1 text chat. Every other feature, group chat, replies and reactions included, is a plugin. Features reach parity with Slack, Discord, Teams and WhatsApp through first-party plugins, all shipping before 1.0.

**Package side vs developer side** ([ADR 5](docs/adr/0005-package-vs-developer-side.md)): the package owns schema, endpoints, types, event contracts and provider interfaces. Whatever runs outside the developer's server (hosting, brokers, media servers, model providers, buckets) is the developer's side. Each vendor area gets one reference adapter.

**Won't in 1.0:** a hosted service, federation, a central app registry, first-party vendor transports beyond SSE and WebSocket, native VoIP push for ringing locked phones.

## 2. Core

Everything in this section ships in `@gnrllybttr/chat` and is always on.

| Area           | Core contains                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| Engine         | Router, `chat.handler`, mount helpers, plugin API, `define*` helpers, config validation               |
| Data           | Schema system, `generate` and `migrate` CLI, Kysely SQL engine with Kysely/Drizzle/Prisma adapters, memory adapter |
| Identity       | Auth resolvers (JWT/JWKS, custom), tenancy                                                             |
| Conversations  | 1:1 direct conversations, exactly one per pair of users per tenant                                     |
| Messages       | `text` kind with a structured body (§6); send, edit, delete for everyone; `clientMessageId`            |
| Read state     | Per-participant `lastReadSeq`, unread counts batched across conversations                              |
| Sync           | Per-conversation `seq` and gap-fill, per-user change feed (`userSeq`), SSE and WebSocket               |
| Server API     | `chat.api.*` (§9), user delete and export (§10)                                                        |
| Safety and ops | Rate limits with pluggable storage, `trustedOrigins` and Origin checks, `logger` option               |

Everything else is a plugin (§12).

## 3. Architecture

**The whole package**

```
 CLIENTS                                                  (developer's frontend)
 +----------------------------------------------------------------------------+
 |  @gnrllybttr/chat/client -> /react (+ React Native) /vue /svelte /solid    |
 |  chat-ui (optional components)     chat-e2ee client side (encrypt/decrypt) |
 +----------------------------------------------------------------------------+
        |  HTTP (RPC): POST /messages/send, GET /sync, ...     ^  SSE / WebSocket
        v                                                      |
 CORE: @gnrllybttr/chat                                    (developer's server)
 +----------------------------------------------------------------------------+
 |  chat.handler -> router -> auth resolver -> tenancy -> rate limits         |
 |                                                  |                         |
 |  chat.api (trusted, server-only) ----------------+                         |
 |                                                  v                         |
 |  +------------------------------ engine ------------------------------+    |
 |  |  1:1 DMs | text + spans | send / edit / delete | clientMessageId   |    |
 |  |  read state + unread counts | seq + gap-fill | userFeed (userSeq)  |    |
 |  |  users.delete / export | import | logger | trustedOrigins / CSRF   |    |
 |  +--------------------------------------------------------------------+    |
 |      ^                              |                        |             |
 |      | plugin API                   v                        v             |
 |      | (schema, endpoints, hooks,   storage adapter       transport        |
 |      |  kinds, spans, events)       (defineAdapter)    (defineTransport)   |
 +------|------------------------------|------------------------|-------------+
        |                              v                        v
        |                    Kysely engine:              in-process (default)
        |                    Postgres, MySQL, SQLite     Redis, Ably, Durable
        |                    + Drizzle / Prisma          Objects, ...
        |                    chat-mongodb / -dynamodb    (developer's side)
        |                    / -firestore
        |
 PLUGINS (opt-in, all ship before 1.0)
 +----------------------------------------------------------------------------+
 |  in @gnrllybttr/chat (subpath exports)                                     |
 |    messaging : groups, channels, replies, threads, reactions, mentions,    |
 |                mention-inbox, role-mentions, presence, notifications,      |
 |                message-controls, mark-unread, message-requests, invites,   |
 |                pins, polls, scheduled-messages, custom-emoji,              |
 |                disappearing-messages                                       |
 |    identity  : sender-identity, system-messages                            |
 |    sharing   : forwarding, links, share-links                              |
 |    reach     : search, push, gifs, link-previews                           |
 |    safety/ops: moderation, audit-log, opentelemetry                        |
 |                                                                            |
 |  separate packages                  talks to (developer's side)            |
 |    chat-better-auth  ............... Better Auth                           |
 |    chat-media  ..................... S3 / R2 / MinIO                       |
 |    chat-ai  ........................ AI SDK -> model providers             |
 |    chat-calls  ..................... LiveKit (clients send media direct)   |
 |    chat-e2ee  (experimental) ....... nothing: server only relays keys      |
 |    chat-apps  ...................... app makers' HTTPS endpoints           |
 +----------------------------------------------------------------------------+

 FOR DEVELOPERS, NOT SHIPPED TO USERS
   chat-testing   memory adapter, fakes, every contract suite
   chat-app-sdk   for people building apps that chat-apps installs
```

**Plugin dependencies** (`A <- B` means B needs A; `definePlugin` checks this at startup)

```
 sender-identity <- system-messages
                 <- chat-ai
                 <- chat-apps

 groups          <- channels
                 <- invites
                 <- role-mentions -> also needs mentions

 mentions        <- mention-inbox
                 <- role-mentions

 optional links (work better together, not required):
   notifications -> read by push, mentions
   system-messages -> used by groups (member.added), chat-calls (call ended)
   chat-media    -> used by chat-ai (files to models), forwarding (reuse files)
   audit-log     -> records chat.api and chat-apps actions
   presence      -> provides @here to mentions
```

- Every request enters through the auth resolver, except `chat.api` calls, which are trusted (§9).
- Core saves first, then publishes. Nothing is published that wasn't saved, except ephemeral events (§8).
- Only enabled plugins talk to outside services, always from the server, except call media, which clients send to the media server directly with tokens the calls plugin mints.

## 4. Runtime, mounting and auth

**Runtime** ([ADR 4](docs/adr/0004-web-standard-runtime.md))

- Core **must** use only web-standard APIs (`Request`, `Response`, `fetch`, Web Crypto, streams) and pass its suites on Node 20+, Bun, Deno and Cloudflare Workers in CI.
- Adapters and plugins that can't run on the edge **must** say so and fail at startup with a clear error.
- `chat.handler` **must** be a plain `(request: Request) => Promise<Response>`. Helpers **must** ship for Next.js (`toNextHandler`), Node/Express (`toNodeHandler`), and WebSocket upgrades on Node, Bun, Deno and Workers.
- Routing is GnrllyBttr Chat's own router ([ADR 1](docs/adr/0001-own-handler-and-router.md)).

**Auth** ([ADR 7](docs/adr/0007-auth.md))

A resolver turns a request into `{ id, tenantId? }` or `null`.

| Resolver    | Package                        | Notes                                                                                     |
| ----------- | ------------------------------ | ----------------------------------------------------------------------------------------- |
| Better Auth | `@gnrllybttr/chat-better-auth` | Session cookie, foreign keys to Better Auth's `user` table, `resolveUsers` filled in       |
| JWT / JWKS  | core                           | Secret or JWKS URL; claims map to `id` and `tenantId`. Docs recipes for Clerk, Supabase, Auth0, Firebase, Auth.js |
| Custom      | `defineResolver`               | Any function returning `{ id, tenantId? }` or `null`                                      |

- `resolveUsers(ids)` **must** be the only way core gets names and avatars.

**Stream auth and request safety** ([ADR 26](docs/adr/0026-security-and-ops.md))

- The same resolver **must** authenticate SSE and WebSocket connections.
- Cookie auth works for SSE as-is. Because `EventSource` can't send headers, the client **must** read SSE with `fetch` when using bearer tokens, and core **must** also offer `POST /stream/ticket`: a single-use ticket valid for 30 seconds, passed as a query parameter.
- `trustedOrigins` **must** be configurable; every mutating request with cookie auth **must** pass an `Origin` check (CSRF). CORS headers come from the same list.

## 5. Tenancy

([ADR 8](docs/adr/0008-core-tenancy.md))

- When the resolver returns `tenantId`, core **must** stamp it on every row and filter every read, count, search and subscription by it, in core and first-party plugins.
- Plugin tables are tenant-scoped by default.
- The storage contract suite **must** prove no read crosses tenants.
- The column is always generated (nullable), so turning tenancy on later needs no schema change.

## 6. Messages

**Body format** ([ADR 18](docs/adr/0018-message-body.md)): plain text plus typed spans.

```ts
{
  text: "hey @Sam see https://x.co",
  spans: [
    { type: "mention", start: 4, end: 8, userId: "u_2" },
    { type: "link", start: 13, end: 26, url: "https://x.co" },
  ],
}
```

- Core span types: `link`, `bold`, `italic`, `strike`, `code`, `codeBlock`. Plugins add theirs (`mention`, `emoji`, ...) with a payload schema.
- Offsets **must** be UTF-16 code units (JavaScript string indices) and **must** be validated: in range, non-overlapping except nesting.
- `text` alone **must** be enough for search, push previews and AI context.
- Client helpers convert Markdown to spans and back.

**Idempotent send** ([ADR 19](docs/adr/0019-idempotent-send.md))

- Every send carries a `clientMessageId`, unique per sender per conversation. A send with an id already saved **must** return the saved message instead of creating another.
- The client shows the message at once with a pending state and swaps in the saved one (optimistic UI).

**Edit and delete**

- Edits set `editedAt`. Deletes are for everyone and leave a placeholder (body removed, `deletedAt` set) so `seq` and counts hold.
- Delete for me, and edit and delete time windows, come from the `message-controls` plugin.

## 7. Schema

Tables are generated from config plus plugins. Developers can rename tables and columns, add columns and add tables; core columns are locked.

**Core tables**

| Table          | Holds                                    | Locked core columns                                                                                   |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `conversation` | Conversations of any type                | `id`, `tenantId`, `type`, `pairKey`, `lastSeq`, `lastActivityAt`                                      |
| `participant`  | Membership and read state                | `conversationId`, `userId`, `tenantId`, `lastReadSeq`                                                 |
| `message`      | Every message, any kind                  | `id`, `conversationId`, `tenantId`, `senderId`, `seq`, `kind`, `text`, `spans`, `clientMessageId`, `editedAt`, `deletedAt`, `createdAt` |
| `userFeed`     | Per-user conversation-level changes      | `userId`, `tenantId`, `userSeq`, `type`, `conversationId`, `payload`, `createdAt`                     |

- `type` is `direct` in core; plugins register more (`group`, `channel`, `ai`).
- Plugins can add tables and add columns to core tables (for example, `replies` adds `message.replyToId`, `threads` adds `message.threadRootId`, `sender-identity` adds `message.senderType`).

**What developers can change**

| Change                       | How                      | Example                               |
| ---------------------------- | ------------------------ | ------------------------------------- |
| Rename a table or column     | `modelName`, `fieldName` | `message` stored as `chat_messages`   |
| Add a column                 | `additionalFields`       | `conversation.orderId: string`        |
| Add a table                  | A plugin's `schema`      | `poll`, `pollVote`                    |
| Change ID format             | `generateId`             | uuid, cuid, serial                    |
| Drop or retype a core column | Not allowed              | Would break ordering and unread counts |

- `additionalFields` **must** support `index: true` and `unique: true`, so linking a conversation to an app record (an order) is fast and can be idempotent ([ADR 15](docs/adr/0015-no-subject-field.md)).

**CLI and upgrades** ([ADR 22](docs/adr/0022-schema-upgrades.md))

- `npx @gnrllybttr/chat generate` **must** write the full schema for the current config and plugins: Drizzle, Prisma or SQL; index and table definitions for MongoDB, DynamoDB and Firestore.
- `npx @gnrllybttr/chat migrate` **must** create the schema on an empty database with the Kysely engine.
- Upgrading between versions: run `generate` again and diff with the ORM's own tools (drizzle-kit, prisma migrate). Every release with schema changes **must** list them in its migration note.

## 8. Sync and realtime

([ADR 5](docs/adr/0005-package-vs-developer-side.md), [ADR 20](docs/adr/0020-user-change-feed.md))

**Two cursors**

| Cursor    | Scope            | Carries                                                                 | Catch-up call                          |
| --------- | ---------------- | ----------------------------------------------------------------------- | -------------------------------------- |
| `seq`     | One conversation | Messages and message-level events (edits, deletes, plugin events)        | `GET /messages/list?after=<seq>`       |
| `userSeq` | One user         | Conversation-level changes: added, removed, updated, unread changed, new last message | `GET /sync?since=<userSeq>` |

- Every stored message event **must** carry its conversation's `seq`; every feed entry its user's `userSeq`. Both are gap-free and strictly increasing.
- On reconnect a client **must** be able to catch up with one `/sync` call, then fetch messages only for conversations it has open.
- `userFeed` entries older than `sync.retention` (default 30 days) may be pruned; a client whose cursor is older gets `SYNC_RESET` and reloads its conversation list.
- Ephemeral events (typing, presence, call ringing, AI token deltas) **must** never be stored or replayed.
- Events are one typed union; plugins add their own.

**Built in**

| Stream    | Notes                                                  |
| --------- | ------------------------------------------------------ |
| SSE       | Default; one-way, auto-reconnect                       |
| WebSocket | Two-way, lower overhead; needs the upgrade helper (§4)  |

- Config: `realtime: { stream: "sse" | "websocket", transport }`. `transport` defaults to in-process fan-out (single long-running server).
- `defineTransport` is `publish(channel, event)` and `subscribe(channel, onEvent)`. Redis, Ably, Pusher, Durable Objects and Supabase Realtime are recipes or community packages.
- The transport contract suite **must** cover delivery, per-conversation ordering and gap-fill after a dropped connection.

## 9. API

([ADR 21](docs/adr/0021-rpc-style-api.md), [ADR 9](docs/adr/0009-trusted-server-api.md))

RPC-style paths that map one-to-one onto `chat.api`: `chat.api.messages.send` is `POST /messages/send`. Reads are `GET`, writes are `POST`. Plugins add endpoints under their own prefix.

**Core endpoints**

| Endpoint                         | `chat.api`                       | Does                                                                 |
| -------------------------------- | -------------------------------- | -------------------------------------------------------------------- |
| `POST /conversations/direct`     | `conversations.direct`           | Find or create the DM with a user                                    |
| `GET /conversations/list`        | `conversations.list`             | The user's conversations by `lastActivityAt`, with unread counts; cursor-paginated |
| `GET /conversations/get`         | `conversations.get`              | One conversation with participants                                   |
| `POST /messages/send`            | `messages.send`                  | Send; idempotent on `clientMessageId`                                |
| `POST /messages/edit`            | `messages.edit`                  | Edit own message                                                     |
| `POST /messages/delete`          | `messages.delete`                | Delete own message for everyone                                      |
| `GET /messages/list`             | `messages.list`                  | Page by `before` or `after` `seq`                                    |
| `POST /read/mark`                | `read.mark`                      | Set `lastReadSeq`                                                    |
| `GET /sync`                      | `sync`                           | Feed entries after `since`                                           |
| `GET /stream`                    | —                                | SSE stream                                                           |
| `GET /ws`                        | —                                | WebSocket upgrade                                                    |
| `POST /stream/ticket`            | —                                | Single-use stream ticket (§4)                                        |
| —                                | `users.delete`, `users.export`   | §10                                                                  |
| —                                | `import`                         | Bulk history import (§10)                                            |

**Errors:** every error is `{ error: { code, message, details? } }` with an HTTP status. Codes: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION`, `CONFLICT`, `RATE_LIMITED` (with `retryAfter`), `LIMIT_EXCEEDED`, `SYNC_RESET`. The client throws typed errors.

**Default limits** (all configurable)

| Limit                          | Default     |
| ------------------------------ | ----------- |
| Message text                   | 10,000 characters |
| Spans per message              | 500         |
| Page size                      | 50, max 200 |
| Request body                   | 1 MB        |

**Trusted server API**

- Every endpoint **must** have a matching `chat.api` function with the same types; `chat.api` **must not** be reachable over HTTP.
- `chat.api` calls take `as: userId` (or `as: "system"` with the `sender-identity` plugin) and `tenantId` when tenancy is on.
- `chat.api` **must** skip `can*` permission hooks but still run `before*` and `after*` hooks, validation and publishing.

## 10. User data and import

([ADR 10](docs/adr/0010-user-data.md))

- `chat.api.users.delete(userId)` **must** remove memberships, read state and feed entries, and anonymize messages (body removed, sender shown as "Deleted user") or hard-delete them, per `users.deleteMode`.
- Every plugin with user-linked tables **must** implement `onUserDelete` and contribute to export; `definePlugin` fails at startup otherwise.
- `chat.api.users.export(userId)` **must** return all of a user's data as JSON.
- `chat.api.import` **must** bulk-load conversations and messages with their original timestamps, assigning `seq` in timestamp order, for moving off Stream, Sendbird or a home-grown system. Imports don't publish realtime events.

## 11. Safety and operations

([ADR 26](docs/adr/0026-security-and-ops.md))

- **Rate limits:** per route and per plugin; `rateLimit.storage` is `memory` (default), `database`, or a `defineRateLimitStore`. Docs **must** say memory doesn't work on serverless or across servers.
- **Logging:** a `logger` option (`debug`, `info`, `warn`, `error`); core never logs message bodies.
- **Scale targets**, verified by a load test in CI before 1.0:

| Target                                         | Value                                    |
| ---------------------------------------------- | ---------------------------------------- |
| Members per conversation                       | 10,000                                   |
| Sustained sends per conversation               | 100 messages/s, `seq` gap-free           |
| Unread counts for a user's conversations       | 500 conversations in one query           |
| Read-state writes                              | Client debounces to at most one per 2 s per conversation |

## 12. Plugins

All first-party plugins ship before 1.0 ([roadmap](docs/roadmap.md)). Small ones are subpath exports of `@gnrllybttr/chat` (`@gnrllybttr/chat/plugins/groups`); big or vendor-tied ones are their own packages.

**Messaging**

| Plugin              | Adds                                                                                                     | Depends on |
| ------------------- | -------------------------------------------------------------------------------------------------------- | ---------- |
| `groups`            | `group` conversations, roles (owner, admin, member), `canManage`, conversation management: add, leave, remove, transfer ownership, rename, avatar, description, archive, pin a chat | — |
| `channels`          | `channel` conversations that members discover and join                                                   | `groups`   |
| `replies`           | `message.replyToId`, quote-reply with a live preview                                                     | —          |
| `threads`           | `message.threadRootId`, `threadState` table, per-thread unread; replies don't count toward the chat's unread unless you follow the thread | — |
| `reactions`         | `reaction` table; any emoji or custom key, idempotent per user                                           | —          |
| `mentions`          | `mention` span and table, `@user`, `@channel` in groups, `unreadMentionCount`, `mention.created`, `canMentionAll` | — |
| `mention-inbox`     | `mentions.list()` across conversations, with its own unread count                                        | `mentions` |
| `role-mentions`     | `@admins` and other role mentions                                                                        | `mentions`, `groups` |
| `sender-identity`   | `message.senderType` (`user`, `bot`, `system`), `chatBot` table; profile lookup checks bots then `resolveUsers` | — |
| `system-messages`   | `system` kind as `{ type, params }` with a fallback `text`; other plugins post through it (for example, `groups` posts `member.added`) | `sender-identity` |
| `presence`          | Typing, online presence, read receipts (with a per-user "hide my receipts" option), `@here`             | —          |
| `notifications`     | Per-participant `notify` (`all`, `mentions`, `none`) and `mutedUntil`; push and mentions read these      | —          |
| `message-controls`  | Delete for me, edit and delete time windows                                                              | —          |
| `mark-unread`       | Mark a conversation or message unread                                                                    | —          |
| `message-requests`  | `canStartDirect` returns allow, request or deny; held DMs in `messageRequest` ([ADR 16](docs/adr/0016-message-requests-plugin.md)) | — |
| `invites`           | Link codes with expiry and use limits; join requests                                                     | `groups`   |
| `pins`, `polls`, `scheduled-messages`, `custom-emoji` | Each its own plugin                                                    | —          |
| `disappearing-messages` | Per-conversation timer; per-tenant retention policy                                                  | —          |

**Sharing**

| Plugin        | Adds                                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------------------- |
| `forwarding`  | Forward to up to 10 chats (configurable) with a note; forward several; share a thread; snapshot plus pointer; "Original deleted" when the source goes (unless `followDeletes: false`); forwarding a forward points at the first original; attachments reused, not copied; `canForward`; `message.forwarded`; rate-limited |
| `links`       | `links.messageUrl()`, "No access" for non-readers, share-sheet helper with copy-link fallback             |
| `share-links` | Public read-only pages behind revocable, expiring tokens; `canCreatePublicLink`; off for encrypted chats  |

**Media, discovery and reach**

| Plugin / package      | Adds                                                                                  | Reference adapter       |
| --------------------- | ------------------------------------------------------------------------------------- | ----------------------- |
| `chat-media`          | `attachment` table; `image`, `file`, `video`, `voice` kinds; signed URLs; a file is removed only when no message uses it | S3-compatible (S3, R2, MinIO) |
| `gifs`                | `gif` kind and picker API                                                             | Klipy                   |
| `link-previews`       | Server-fetched titles and images; **must** block private and internal addresses       | —                       |
| `search`              | Message search that respects tenancy and read access                                  | Database full-text      |
| `push`                | Push on new messages and mentions, honouring `notifications`                          | Web Push                |
| `moderation`          | Block, mute user, report, ban, content-filter hook; in encrypted chats the reporter's client attaches the decrypted message | — |
| `audit-log`           | `auditLog` table: actor, action, target, tenant; records `chat.api` and app actions   | —                       |
| `opentelemetry`       | Spans for requests, storage calls and deliveries                                      | —                       |

**Big features** (§13): `chat-ai`, `chat-calls`, `chat-e2ee`, `chat-apps`.

**Vendor interfaces** ([ADR 5](docs/adr/0005-package-vs-developer-side.md)): each has a contract suite in `chat-testing`.

| Interface              | Reference adapter                | Elsewhere (recipes or community)     |
| ---------------------- | -------------------------------- | ------------------------------------ |
| `defineAdapter`        | Kysely SQL engine                | —                                    |
| `defineTransport`      | In-process                       | Redis, Ably, Pusher, Durable Objects |
| `defineStorageProvider`| S3-compatible                    | Local disk, GCS                      |
| `defineGifProvider`    | Klipy                            | Giphy                                |
| `definePushProvider`   | Web Push                         | FCM, APNs, Expo                      |
| `defineSearchProvider` | Database full-text               | Meilisearch, Typesense, OpenSearch   |
| `defineCallProvider`   | LiveKit                          | Daily, Cloudflare Calls              |
| `defineModelProvider`  | AI SDK                           | —                                    |
| `defineRateLimitStore` | Memory, database                 | Redis                                |

## 13. Big features

### AI chats (`chat-ai`, needs `sender-identity`)

Nothing AI runs until the developer declares an assistant. Each assistant is a `chatBot`, so AI chats reuse history, unread counts, search and sharing.

```ts
createChat({
  plugins: [
    senderIdentity(),
    ai({
      assistants: {
        helper: { model: anthropic("<model>"), system: "You help customers.", tools: { lookupOrder } },
      },
      allow: { directChats: true, groupMentions: true },
    }),
  ],
});
```

| Feature             | Requirement                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| 1:1 AI chat         | `ai.startChat("helper")` opens an `ai` conversation                                               |
| AI in groups        | With `groups` and `mentions`: `@helper` asks the assistant; it sees only that conversation        |
| Streaming           | Ephemeral `message.delta` events; the finished reply is saved once as a normal message           |
| Stop and regenerate | Stop cancels the model call; regenerate keeps earlier answers as versions                        |
| Tools               | Developer-defined tools with optional user approval; MCP server tools                            |
| Files and images    | With `chat-media`: attachments go to models that accept them                                     |
| Rich replies        | `kind: "ai"` holds typed parts: text, reasoning, tool calls, sources                             |
| Context             | Recent messages trimmed to the model's limit; `buildContext` hook                                |
| Titles and feedback | Optional auto titles; thumbs up or down stored for the developer                                 |
| Usage               | `aiUsage` table; `beforeAiRequest` enforces quotas and picks a model per plan                    |

- Model calls **must** run server-side; only declared assistants can be used.
- Generation **must** continue if the client disconnects; the reply arrives through sync.
- On serverless, `ai.runGeneration(jobId)` runs a generation from a background job.
- `chat-testing` ships a fake model that plays scripted streams.

### Calls (`chat-calls`)

The plugin owns ringing, permissions and call history; media runs through the call provider (LiveKit by default).

1. `calls.start` checks `canCall` and `beforeCallStart`, creates a `call` row and sends an ephemeral `call.ringing`.
2. Callees accept or decline; no answer within the timeout marks it missed.
3. On accept, the plugin mints a provider token for that room, only for participants.
4. Clients join the room directly; media never touches the developer's server.
5. Provider webhooks report joins and leaves; when the room empties the plugin closes the call, posts a `call` message (via `system-messages` when enabled), and runs `onCallEnd`.

**1.0 covers:** audio, video, mute, camera toggle, screen share, configurable group size, recording. **Won't:** ringing locked phones.

### End-to-end encryption (`chat-e2ee`)

([ADR 11](docs/adr/0011-e2ee-experimental.md))

- Ships labelled **experimental and unaudited** (docs and startup warning) until an independent review is published.
- MLS (RFC 9420) via ts-mls, opt-in per conversation. Each device is its own MLS member; key bundles and handshake messages are relayed, never read, by the server.
- The server sees ciphertext bodies and attachments; ids, senders, `seq`, timestamps and mentioned user ids stay readable, so unread counts, sync and mention alerts work.
- Off in encrypted conversations: server search, link previews, content-filter hooks, AI assistants, apps, public links.
- Forwarding decrypts on the client and re-encrypts for the target. Reports attach the decrypted message from the reporter's client.
- Calls take media keys from the MLS group.
- Core **must** stay E2EE-ready: `text` may be opaque ciphertext with `spans` empty.

### Apps (`chat-apps`, `chat-app-sdk`, needs `sender-identity`)

([ADR 12](docs/adr/0012-apps-protocol-now-registry-later.md))

1. **Manifest:** JSON at an HTTPS URL naming the app, icon, scopes, events, slash commands, message-menu actions and endpoint.
2. **Install:** an admin installs by manifest URL and approves scopes; the plugin creates a `chatBot` and a scoped token per install (per tenant when set).
3. **Events:** signed with HMAC-SHA256 over a timestamp and body; the SDK rejects anything over 5 minutes old.
4. **Actions:** the app calls the API with its token.
5. **UI:** JSON blocks (text, images, buttons, menus, forms) the client renders; clicks go back as events.
6. **Uninstall:** revokes the token, removes the bot everywhere, notifies the app.

- `apps: { enabled, allow, deny, maxScopes }` is off by default; `canInstallApp` decides who installs.
- Deliveries go through an outbox with retry and backoff; on serverless a cron or queue runs `apps.deliver()`.
- Each install is rate-limited; app actions are recorded by `audit-log` when enabled.
- **Won't in 1.0:** a central registry, publisher verification, delisting, payments.

## 14. Type safety

- `chat.$Infer.*` types **must** include `additionalFields` and every enabled plugin's fields.
- `createChatClient<typeof chat>()` **must** pick up every field, endpoint, event, message kind and span type with no codegen.
- Inputs **must** be validated at the edge with any Standard Schema library (Zod, Valibot, ArkType).

## 15. Extending

```ts
const polls = () =>
  definePlugin({
    id: "polls",
    dependsOn: [],
    schema: { poll: { fields: { question: { type: "string" } } } },
    endpoints: { vote: createChatEndpoint("/polls/vote", { method: "POST" }, handler) },
    messageKinds: { poll: pollPayloadSchema },
    events: { "poll.voted": voteEventSchema },
    hooks: { after: [{ matcher, handler }] },
    onUserDelete: async (ctx, userId) => { /* ... */ },
    rateLimit: [{ pathMatcher, limit: 10, window: 60 }],
  });
```

**App-level hooks:** `canRead`, `canWrite` (plus `canManage`, `canForward`, `canMentionAll`, `canCall`, `canInstallApp` from their plugins), `beforeMessageSend`, `afterMessageMutation`, `beforeCallStart`, `onCallEnd`, `beforeAiRequest`, `databaseHooks.<table>.create/update/delete`.

| Extension          | Helper            | Must pass                                                                      |
| ------------------ | ----------------- | ------------------------------------------------------------------------------ |
| Plugin             | `definePlugin`    | Startup checks: unique id, declared dependencies present, no route/table/field clashes, `onUserDelete` when user-linked |
| Database adapter   | `defineAdapter`   | Storage contract suite                                                         |
| Transport          | `defineTransport` | Transport contract suite                                                       |
| Vendor providers   | `define*`         | That interface's suite (§12)                                                   |
| Auth resolver      | `defineResolver`  | Returns `{ id, tenantId? }` or `null`                                          |

- `npx @gnrllybttr/chat plugin init` scaffolds a plugin with tests, Biome and the coverage gate.
- Plugins list `@gnrllybttr/chat` as a peer dependency. Published plugins use the npm keyword `gnrllybttr-chat-plugin`.
- **Design rules:** one plugin per feature, one adapter per database; features arrive as plugins, never core edits; every adapter and transport passes the same suite; core depends on interfaces, never vendor SDKs; plain objects and functions, no classes, decorators or DI container.

## 16. Clients and UI

- `@gnrllybttr/chat/client`: framework-agnostic, on nanostores; optimistic sends, sync on reconnect, typed errors.
- Bindings: `/react` (also React Native and Expo), `/vue`, `/svelte`, `/solid`.
- `@gnrllybttr/chat-ui`: optional unstyled components, including the blocks renderer. **Must** meet WCAG 2.1 AA: full keyboard use, focus management, screen-reader announcements for new messages.

## 17. Repo, release and testing

**Repo:** one Bun and Turborepo monorepo at [GnrllyBttr/GnrllyBttrChat](https://github.com/GnrllyBttr/GnrllyBttrChat): `packages/*`, `apps/docs`, `examples/*`.

| Job                         | Tool                                          |
| --------------------------- | --------------------------------------------- |
| Installs, workspaces, tests | Bun, with catalogs                            |
| Tasks and caching           | Turborepo                                     |
| Lint and format             | Biome v2                                      |
| Type check                  | TypeScript `strict`                           |
| Build                       | tsdown (ESM, `.d.ts`, publint, attw)          |
| Dead code                   | Knip                                          |
| Git hooks and commits       | Lefthook, commitlint (Conventional Commits)   |
| Versions and changelogs     | Changesets                                    |
| Publishing                  | npm trusted publishing                        |
| Dependency updates          | Renovate                                      |

**Release** ([ADR 25](docs/adr/0025-versioning.md))

- All first-party packages share one version (a Changesets `fixed` group) until 1.0.
- During 0.x a minor release may break; every one ships a migration note covering API and schema changes. Patches never break.
- Merging the Changesets PR packs each package with `bun pm pack` and publishes with `npm publish`.
- `main`: PRs only, all checks required, linear history. Security: CodeQL, secret scanning, Dependabot alerts, OpenSSF Scorecard.

**Testing** ([ADR 13](docs/adr/0013-coverage.md))

1. Test first; a bug fix starts with a failing test.
2. Unit tests with the memory adapter, fake transport and fake clock.
3. Contract suites for every adapter, transport and provider, against real services in CI: Postgres, MySQL, SQLite, MongoDB, DynamoDB Local, Firestore emulator, LiveKit dev server.
4. Integration tests on the real handler over SSE and WebSocket, on every runtime in §4.
5. A load test for the scale targets (§11) before 1.0.
6. Example apps, each smoke-tested in CI: SaaS buyer-seller (Next.js, Better Auth, Postgres), 1:1 messenger (Expo), social group chat (SvelteKit), team chat (React, Hono).

- Coverage: 100% lines, functions and branches for core, adapters and first-party plugins. A second, branch-measuring runner runs alongside Bun; tests import test APIs through a shim so both runners use the same files.
- Ignores need a one-line reason (generated files, provably unreachable branches); CI reports new ones on each PR.

## 18. Open items

- [ ] **npm scope:** create the `@gnrllybttr` org before the first publish.
- [ ] **Branch coverage runner:** pick it (for example, Vitest with V8 coverage) and prove the shim.
- [ ] **E2EE review:** who reviews `chat-e2ee`, at what cost, and when.
- [ ] **App registry (post-1.0):** repo, review process, reviewers.
