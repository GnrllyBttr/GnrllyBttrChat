# GnrllyBttrChat: Design Spec

Final spec for GnrllyBttrChat 1.0: user and AI chats, sharing, calls and a Marketplace, built test-first in a Bun and Turborepo monorepo. Decisions, and the items still open, are at the end.

## Goals

GnrllyBttrChat is an MIT-licensed, open-source TypeScript chat backend that runs inside your app and stores everything in your database, the way Better Auth does for auth. It keeps Chatpack's core model and closes its gaps:

| Gap in Chatpack today                       | GnrllyBttrChat answer                                                                                    |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Fixed schema, no generate step              | Schema built from config and plugins, with `generate` and `migrate` CLI commands                         |
| Auth is a bare `{ id }` resolver            | First-class Better Auth integration, plus resolvers for any other auth                                   |
| Custom data only in a `metadata` JSON field | Typed extra columns that flow into server and client types                                               |
| Live features stop working on serverless    | SSE and WebSocket both built in, plus hosted pub/sub for serverless                                      |
| No threads, push, calls or encryption       | Threads, push hooks, GIF and meme providers, 1:1 and group voice and video, opt-in end-to-end encryption |
| No sharing, AI chat or app ecosystem        | Forwarding and links, opt-in AI chats, and a Marketplace any GnrllyBttrChat app can install from         |

Not in 1.0: a hosted service, or federation between servers.

## Built in vs first-party plugins

Core covers what every chat needs; everything else is a first-party plugin the developer turns on. Small plugins ship inside `@gnrllybttr/chat`; bigger ones are their own packages, named in brackets.

| Area          | Built in (core)                                                            | First-party plugins and packages (opt-in)                               |
| ------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Conversations | DMs, groups, channels, roles                                               | Invites and join requests                                               |
| Messages      | Send, edit, delete, replies, threads, emoji reactions                      | Pins, polls, scheduled messages, custom emoji packs                     |
| Mentions      | `@user`, `@here`, `@channel`, mention inbox, @ badges                      | `@role` mentions                                                        |
| Read state    | Unread counts, read position                                               | Typing, presence, read receipts                                         |
| Sharing       | Forward, forward several, share a thread, copy link, device share sheet    | Public web links                                                        |
| Media         | None                                                                       | Files, images, video, voice notes; GIFs, stickers, memes; link previews |
| Search        | None                                                                       | Database full-text, or Meilisearch and Typesense                        |
| Notifications | Events and hooks                                                           | Push: Web Push, FCM, APNs, Expo                                         |
| Safety        | Permission hooks, rate limits                                              | Moderation: block, mute, report, ban, content filters                   |
| AI            | None                                                                       | AI chats (`chat-ai`)                                                    |
| Calls         | None                                                                       | Voice and video, 1:1 and group (`chat-livekit`)                         |
| Encryption    | None                                                                       | End-to-end encryption (`chat-e2ee`)                                     |
| Ecosystem     | Plugin API, `define*` helpers                                              | Marketplace host (`chat-apps`), app maker SDK (`chat-app-sdk`)          |
| Auth          | Resolver interface; Clerk, Auth.js, Supabase, Firebase, JWT resolvers      | Better Auth integration (`chat-better-auth`)                            |
| Data          | Schema system, CLI, SQL adapters (Kysely, Drizzle, Prisma), memory adapter | MongoDB, DynamoDB, Firestore adapters                                   |
| Realtime      | SSE, WebSocket, ordering and gap-fill                                      | Redis fan-out, hosted pub/sub (`chat-redis`, `chat-ably`, ...)          |
| Dev tools     | Typed client, React, Vue and Svelte hooks                                  | UI components (`chat-ui`), test kit (`chat-testing`)                    |

## Architecture

One `gnrllyBttrChat()` config produces one request handler inside your server. Chat data goes through a storage adapter into your own database; only plugins you enable, such as calls, AI and Marketplace apps, talk to outside services.

&#91;embedded content: GnrllyBttrChat architecture · server parts, transports, LiveKit\]

Requests enter through Auth, the core publishes saved events to the realtime transport, and clients join LiveKit rooms directly with tokens the calls plugin mints. AI models and Marketplace apps are reached from the server, never from the browser.

## Auth

GnrllyBttrChat has its own handler, adapters and CLI and never builds on Better Auth's core. Auth is a resolver that turns a request into a user `{ id }`; Better Auth gets the most complete one.

| Auth                      | Setup                                                                | What you get                                                                                                           |
| ------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Better Auth (first-class) | `auth: betterAuthResolver(auth)` from `@gnrllybttr/chat-better-auth` | Reads the Better Auth session cookie, real foreign keys to its `user` table, names and avatars filled in automatically |
| Any other auth            | `auth: resolver`                                                     | Prebuilt resolvers for Clerk, Auth.js, Supabase, Firebase and plain JWT, or your own                                   |

- **Independent:** GnrllyBttrChat versions on its own and needs no Better Auth install unless you choose it.
- **Live stream:** served by GnrllyBttrChat's own handler, so Better Auth's GET-and-POST-only endpoint rule never applies.
- **User profiles:** a `resolveUsers(ids)` hook fills names and avatars from your user store; automatic with Better Auth.
- **Multi-tenant apps:** a resolver may also return `tenantId`, the customer org a user belongs to; Marketplace installs are scoped by it.

## Schema

Tables are generated from config plus plugins, Better Auth style. Devs can rename, add columns and add tables, but core columns are locked because ordering and read state depend on them.

**Default tables**

| Table          | Holds                                                       | Locked core columns                                                                                        |
| -------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `conversation` | DMs, groups, channels                                       | `id`, `type`, `lastSeq`, `lastActivityAt`                                                                  |
| `participant`  | Membership, role, read state                                | `conversationId`, `userId`, `role`, `lastReadSeq`                                                          |
| `message`      | Every message, any kind                                     | `id`, `conversationId`, `senderId`, `seq`, `kind`, `threadRootId`, `replyToId`, `forwardOfId`, `deletedAt` |
| `reaction`     | Emoji reactions                                             | `messageId`, `userId`, `emoji`                                                                             |
| `mention`      | Who a message names; a seenAt time drives the mention inbox | `messageId`, `userId`                                                                                      |
| `attachment`   | Files, images, voice notes                                  | `messageId`, `storageKey`, `mime`, `size`                                                                  |

Plugins add their own tables (calls, invites, moderation, push tokens, polls, custom emoji, encryption keys, public share links, Marketplace installs, AI usage).

**What devs can change**

| Change                       | How                      | Example                                |
| ---------------------------- | ------------------------ | -------------------------------------- |
| Rename a table or column     | `modelName`, `fieldName` | `message` stored as `chat_messages`    |
| Add a column                 | `additionalFields`       | `message.priority: number`             |
| Add a table                  | A plugin's `schema`      | `poll`, `pollVote`                     |
| Change ID format             | `generateId`             | uuid, cuid, serial                     |
| Drop or retype a core column | Not allowed              | Would break ordering and unread counts |

```ts
gnrllyBttrChat({
  database: drizzleAdapter(db, { provider: "pg" }),
  schema: {
    message: {
      modelName: "chat_messages",
      additionalFields: { priority: { type: "number", required: false } },
    },
  },
});
```

**CLI:** `npx @gnrllybttr/chat generate` writes Drizzle, Prisma or SQL; `npx @gnrllybttr/chat migrate` applies it with the built-in adapter. For MongoDB, DynamoDB and Firestore it writes index and table definitions instead. The same CLI runs whichever auth you use.

## Databases

GnrllyBttrChat ships its own adapters for SQL, MongoDB, DynamoDB and Firestore. Each adapter has two layers: plain reads and writes, and five chat-specific atomic operations.

- **Generic layer:** `create`, `findOne`, `findMany`, `update`, `delete`, `count`, `transaction`. Same shape as Better Auth's adapter contract, so its adapter authors feel at home, but GnrllyBttrChat's own code ([Better Auth: create a DB adapter](https://better-auth.com/docs/guides/create-a-db-adapter)).
- **Chat layer:** `appendMessage` (next `seq` and insert in one atomic step), `createDirectOnce` (one DM per pair), `addOnce` (idempotent reactions and members), `countUnread` (batched), `search` (optional).

**How each database handles the hard parts**

| Database  | Message order (`seq`)                                                               | One DM per pair                                         | Live feed source       | Watch out for                                                                                                                                                              |
| --------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MongoDB   | `$inc` on the conversation doc plus insert, in one transaction                      | Unique partial index on `pairKey`                       | Change streams         | Transactions and change streams need a replica set; a single-node one is enough                                                                                            |
| DynamoDB  | `TransactWriteItems`: bump the counter and put the message; all succeed or all fail | Conditional put (`attribute_not_exists`) on a pair item | DynamoDB Streams       | Only key and index lookups are cheap: one index per access path, per-participant unread counters, search via an OpenSearch adapter                                         |
| Firestore | Transaction on the conversation doc's counter; messages in a subcollection          | Pair key as the doc ID, created only if absent          | `onSnapshot` listeners | A single doc's update rate is limited and contends when busy; a collection with a sequentially indexed field caps at 500 writes/s. Admin SDK does not run on edge runtimes |

Sources: [MongoDB change streams](https://www.mongodb.com/docs/master/changeStreams/), [DynamoDB TransactWriteItems](https://docs.aws.amazon.com/goto/WebAPI/dynamodb-2012-08-10/TransactWriteItems), [Firestore best practices](https://docs.cloud.google.com/firestore/native/docs/best-practices).

**Better Auth on the same database**

| Database  | Better Auth today                                                                                                                                                      | Plan                                                                                                                                                          |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MongoDB   | Official adapter ([Better Auth: database](https://better-auth.com/docs/concepts/database))                                                                             | Auth uses Better Auth's adapter, chat uses `@gnrllybttr/chat-mongodb`, same database                                                                          |
| Firestore | Community [better-auth-firestore](https://socket.dev/npm/package/better-auth-firestore) (MIT), built to pair with `better-auth-firebase-auth` for Firebase sign-in     | Same split: their adapter for auth, `@gnrllybttr/chat-firestore` for chat                                                                                     |
| DynamoDB  | No Better Auth adapter found. Auth.js has an official [single-table one](https://authjs.dev/getting-started/adapters/dynamodb), and Auth.js is now part of Better Auth | Ship `@gnrllybttr/better-auth-dynamodb`, a separate Better Auth adapter, so auth can live on DynamoDB next to chat; submit it to Better Auth's community list |

- **DynamoDB catch:** Better Auth looks up users by email, sessions by token and accounts by provider, so each needs its own index (the Auth.js adapter uses one shared `GSI1`). Any other filter would mean a full-table scan, so the adapter should refuse it loudly.
- **Firebase Auth** already works in standalone mode through the auth resolver.

## Type safety

One config is the source of truth: every extra column, plugin table and message kind shows up in server and client types with no codegen step.

- **Server:** `chat.$Infer.Message`, `chat.$Infer.Conversation` include your `additionalFields` and plugin fields, the way `auth.$Infer.Session` works in Better Auth.
- **Client:** `createChatClient<typeof chat>()` picks up every server field, plugin endpoint and event, the way Better Auth clients use `$InferServerPlugin` ([Better Auth plugins](https://better-auth.com/docs/concepts/plugins)).
- **Message kinds:** `message.kind` is a typed union (`text`, `image`, `gif`, `voice`, `call`, `system`, plus plugin kinds like `poll`), each with its own payload type.
- **Events:** realtime events are one typed union, so `on("message.created", ...)` knows its payload, including plugin events.
- **Runtime checks:** extra fields are validated at the API edge with any Standard Schema library (Zod, Valibot, ArkType), so types and validation never drift.

## Realtime

Every message is saved first, then published, and carries a per-conversation `seq` number, so a client that drops off can ask for everything after its last `seq` and miss nothing. How events travel is a swappable transport.

SSE and WebSocket both ship in core as first-class options; the dev picks with `realtime: { transport: "sse" | "websocket" }`, and any transport below can be swapped in.

| Transport                                                                    | Use when                                       | Notes                                                                     |
| ---------------------------------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------- |
| SSE (used when unset)                                                        | One long-running server                        | One-way stream from server to browser; auto-reconnect built into browsers |
| WebSocket                                                                    | Mobile apps, heavy typing and presence traffic | Two-way, lower overhead per event                                         |
| Redis fan-out                                                                | Several servers                                | Pairs with SSE or WebSocket                                               |
| Hosted pub/sub (Ably, Pusher, Cloudflare Durable Objects, Supabase Realtime) | Serverless (Vercel, Lambda)                    | Server publishes; browser subscribes directly with a scoped token         |
| Polling                                                                      | Last resort                                    | No typing or presence                                                     |

- **Ephemeral events** (typing, presence, call ringing) are never stored or replayed.
- **Fix over Chatpack:** on serverless, Chatpack falls back to polling and loses typing, presence and receipts ([Chatpack guide](https://raw.githubusercontent.com/chddaniel/chatpack/main/llms.txt)). The hosted pub/sub transport keeps them working.

## Chat features

Core ships what every chat needs; everything else is an opt-in plugin so apps only pay for what they use.

| Feature                           | Where  | Notes                                                                             |
| --------------------------------- | ------ | --------------------------------------------------------------------------------- |
| Send, edit, delete                | Core   | Deletes leave a placeholder so order and counts hold                              |
| Reply to a message                | Core   | Quote-reply with a live preview of the original                                   |
| Threads                           | Core   | Side conversation under a message, with its own unread count                      |
| Emoji reactions                   | Core   | Any emoji or custom key                                                           |
| Mentions                          | Core   | `@user`, `@here`, `@channel`, plus a mention inbox and badges (details below)     |
| Read state and unread counts      | Core   | Stored per participant                                                            |
| DMs, groups, channels             | Core   | Roles: owner, admin, member                                                       |
| Typing, presence, read receipts   | Plugin | Live only, never stored                                                           |
| Files, images, video, voice notes | Plugin | Storage adapter: S3, R2, local disk; signed URLs                                  |
| GIFs, stickers, memes             | Plugin | Provider adapter: Giphy or Klipy                                                  |
| Custom emoji packs                | Plugin | Per workspace or per conversation                                                 |
| Link previews                     | Plugin | Server fetches page titles and images, with a guard against internal-network URLs |
| Pins, polls, scheduled messages   | Plugin | Each its own plugin                                                               |
| Search                            | Plugin | Database full-text by default; adapter for Meilisearch or Typesense               |
| Push notifications                | Plugin | Adapters: Web Push, FCM, APNs, Expo                                               |
| Invites and join requests         | Plugin | Link codes with expiry and use limits                                             |
| Moderation                        | Plugin | Block, mute, report, ban, content filter hook                                     |
| End-to-end encryption             | Plugin | Opt-in per conversation; see End-to-end encryption below                          |
| Forward, copy link, share         | Core   | Public web links are a plugin; see Sharing and forwarding                         |
| AI chats                          | Plugin | Off until the developer declares an assistant; see AI chats                       |

**GIF note:** Google shut down the Tenor API on June 30, 2026, and Discord, WhatsApp and Bluesky moved to Klipy ([PC Gamer](https://www.pcgamer.com/hardware/google-kills-yet-another-api-shutting-out-all-non-google-platforms-from-favoured-gif-database/)). No Tenor adapter.

**Mentions in detail**

Typing `@` autocompletes people in the conversation, and the message stores their user ids beside the text, so a rename never breaks an old mention.

| Mention          | Reaches                             | Notes                                                                                                    |
| ---------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `@user`          | That person                         | Must be a participant; mentioning someone outside offers to add them, if the sender can manage the group |
| `@here`          | Members online now                  | Groups and channels only; needs the presence plugin                                                      |
| `@channel`       | Every member                        | Groups and channels only                                                                                 |
| `@role` (plugin) | Members with a role, e.g. `@admins` | Opt-in                                                                                                   |

- **Who can mass-mention:** a `canMentionAll` hook decides who may use `@here` and `@channel`, so big channels don't get spammed.
- **Mention inbox:** `mentions.list()` returns every message that names me, across conversations, with its own unread count. Chatpack stores mentions but keeps no inbox and notifies nobody ([Chatpack guide](https://raw.githubusercontent.com/chddaniel/chatpack/main/llms.txt)).
- **Badges:** each conversation returns `unreadMentionCount` next to `unreadCount`, for a separate @ badge.
- **Notifications:** mentioned users get a `mention.created` event, and `afterMessageMutation` receives `mentionedIds`, so push can target them even in a muted conversation (unless they turned mentions off there).
- **Edits:** adding a mention on edit notifies that person once; removing one takes it out of their inbox.

## Sharing and forwarding

Any message, including a thread reply, can be forwarded to other chats, linked to, or handed to another app. Forwarding and links are core; public web links are an opt-in plugin.

| Action                             | What happens                                                                                                                                                 | Who can                                                                                                        |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Forward                            | Posts into each chosen chat (10 by default, configurable) with an optional note and a "Forwarded" card: original text, sender, time, attachments             | Anyone who can read the source and write in the target; `canForward` can block it, e.g. for confidential chats |
| Forward several                    | Pick several messages, from the main chat or a thread, and forward them as one bundle                                                                        | Same                                                                                                           |
| Share a thread                     | Forwards the root message with a link to the whole thread                                                                                                    | Same                                                                                                           |
| Copy link                          | A permalink built by `links.messageUrl(conversationId, messageId)`; opens the app scrolled to that message                                                   | Content shows only to people who can read the chat; others see "No access"                                     |
| Public link (`share-links` plugin) | Read-only web page behind an unguessable token, with optional expiry; revocable                                                                              | Off by default; `canCreatePublicLink` decides who                                                              |
| Share to another app               | Client helper opens the device share sheet with the link and text ([MDN: navigator.share](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share)) | Not in every browser, so it falls back to copying the link                                                     |
| Send to a Marketplace app          | Message menu actions such as "Create task" (see Marketplace)                                                                                                 | Where an admin installed that app                                                                              |

- **Snapshot plus link:** a forward stores a copy of the original and a pointer to it, so it reads right after edits. "View original" opens the source for people who can read it.
- **Deletes carry over:** if the original is deleted or removed by a moderator, its forwards show "Original deleted". Set `forward.followDeletes: false` to keep them.
- **No duplicate files:** forwards reuse the stored attachment; each viewer gets their own signed URL. A file is removed only when no message uses it.
- **No stacked labels:** forwarding a forward points at the first original.
- **Encrypted chats:** the client decrypts the source and re-encrypts the forward for the target. Public links and server previews are off for these messages.
- **Events and limits:** `message.forwarded` fires for plugins and push; forwarding is rate-limited to stop spam.

## Voice and video calls

GnrllyBttrChat's opt-in calls plugin owns ringing, permissions and call history; audio and video run through a swappable media server, LiveKit by default, for both 1:1 and group calls.

- **Why a media server even for 1:1:** group calls need an SFU (a server that forwards each person's stream to everyone else), and direct 1:1 calls still need relay servers to get through firewalls. One path for both is simpler.
- **Why LiveKit:** Apache 2.0 server, self-host as a single binary or Docker, or use LiveKit Cloud. It authenticates joins with JWT access tokens, sends webhooks, includes TURN relays, and supports recording through Egress ([LiveKit README](https://github.com/livekit/livekit/blob/master/README.md)).
- **Adapter interface:** `createRoom`, `mintToken`, `endRoom`, `handleWebhook`, so Daily or Cloudflare can be added later.

**Call flow**

1. Caller hits `calls.start` in a conversation. GnrllyBttrChat checks `canCall`, creates a `call` row and sends a `call.ringing` event to the other participants.
2. Callees accept or decline. No answer within the ringing timeout marks it missed.
3. On accept, GnrllyBttrChat mints a LiveKit token for that call's room, only for conversation participants.
4. Clients join the room directly; media never touches the GnrllyBttrChat server.
5. LiveKit webhooks report joins and leaves. When the room empties, GnrllyBttrChat closes the call and posts a `call` message with its duration.

**Scope:** 1.0 covers audio, video, mute, camera toggle, screen share and a configurable group size. Ringing a locked phone needs native VoIP push (CallKit on iOS), so it waits for a later phase.

## End-to-end encryption

Encryption ships in 1.0 as an opt-in plugin, `@gnrllybttr/chat-e2ee`, switched on per conversation; apps that need server search or AI replies leave it off.

- **Protocol:** MLS (Messaging Layer Security, the IETF standard for encrypted group chat, RFC 9420) via [ts-mls](https://cdn.jsdelivr.net/npm/ts-mls@1.6.2/README.md), an MIT TypeScript library.
- **What the server sees:** bodies and attachments are ciphertext. Conversation ids, senders, order, timestamps and mentioned user ids stay readable, so unread counts, gap-fill and mention alerts still work.
- **What turns off there:** server search, link previews, content-filter hooks and server-side AI replies. Search runs on the device instead.
- **Devices:** each device joins the MLS group as its own member; the plugin adds tables for public key bundles and handshake messages, which the server only relays. A new device can't read older messages unless the app adds key backup.
- **Calls:** LiveKit supports end-to-end encrypted media ([LiveKit README](https://github.com/livekit/livekit/blob/master/README.md)), so calls in an encrypted conversation can take their keys from the MLS group.
- **Risk:** ts-mls has had no formal security audit, and its authors advise an independent review before security-critical use ([ts-mls](https://cdn.jsdelivr.net/npm/ts-mls@1.6.2/README.md)). Budget one before 1.0.

## AI chats

User-to-AI chats, like Claude or ChatGPT, ship as an opt-in plugin, `@gnrllybttr/chat-ai`: nothing AI runs until the developer adds it and declares an assistant. Each assistant is a bot user, so AI chats reuse history, unread counts, search, sharing and mentions.

```ts
gnrllyBttrChat({
  plugins: [
    ai({
      assistants: {
        helper: {
          model: anthropic("<model>"),
          system: "You help customers.",
          tools: { lookupOrder },
        },
      },
      allow: { directChats: true, groupMentions: true },
    }),
  ],
});
```

| Feature                           | How                                                                                                                                                                                                                      |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1:1 AI chat                       | A DM with an assistant; `ai.startChat("helper")`                                                                                                                                                                         |
| AI in group chats                 | `@helper` in a group asks the assistant; it sees only that chat                                                                                                                                                          |
| Streaming                         | Tokens arrive as `message.delta` events over SSE or WebSocket; the finished reply is saved once as a normal message                                                                                                      |
| Stop, regenerate, edit and resend | Stop cancels the model call; regenerate keeps earlier answers as versions to flip between                                                                                                                                |
| Models                            | Any model through the AI SDK, which puts OpenAI, Anthropic and other providers behind one interface ([AI SDK: streamText](https://ai-sdk.dev/docs/reference/ai-sdk-core/stream-text)), or your own `defineModelProvider` |
| Tools                             | Developer-defined tools, with optional user approval before one runs; MCP (Model Context Protocol) servers' tools too ([AI SDK: tools](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling.md))                   |
| Files and images                  | Chat attachments go to models that accept them                                                                                                                                                                           |
| Rich replies                      | `message.kind: "ai"` holds typed parts: text, reasoning, tool calls, sources                                                                                                                                             |
| Context                           | Recent messages trimmed to the model's limit; a `buildContext` hook can summarise or add your own data                                                                                                                   |
| Titles and feedback               | Optional auto titles for new AI chats; thumbs up or down on replies, stored for the developer                                                                                                                            |
| Usage                             | Tokens per user and chat in an `aiUsage` table; `beforeAiRequest` enforces quotas and picks a model per plan                                                                                                             |

- **Keys stay on the server:** model calls run server-side; API keys never reach the browser.
- **Users can't pick arbitrary models:** only assistants the developer declared can be used.
- **Closed tab:** generation keeps going and the reply is saved; reopening fetches it through the normal gap-fill.
- **Serverless:** long answers can outlast function time limits, so run generation in a background job or a long-running route.
- **Encrypted chats:** the server can't read them, so assistants can't join them in 1.0.
- **Hosted AI bots:** an AI bot can also be a Marketplace app, using the same events and API as any app.
- **Testing:** `@gnrllybttr/chat-testing` ships a fake model that plays scripted streams, so AI features hit 100% coverage with no paid API calls.

## Plugins and hooks

A GnrllyBttrChat plugin has the same shape as a Better Auth plugin, plus chat-only keys for events and message kinds, so anyone who has written one can write the other.

```ts
const polls = () =>
  definePlugin({
    id: "polls",
    schema: { poll: { fields: { question: { type: "string" } } } }, // picked up by generate
    endpoints: {
      vote: createChatEndpoint("/polls/vote", { method: "POST" }, handler),
    },
    messageKinds: { poll: pollPayloadSchema }, // adds "poll" to message.kind
    events: { "poll.voted": voteEventSchema }, // adds a typed realtime event
    hooks: { after: [{ matcher, handler }] },
    rateLimit: [{ pathMatcher, limit: 10, window: 60 }],
  });
```

**Hooks every app can set**

| Hook                                                        | Runs                                  | Typical use                                           |
| ----------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------- |
| `auth`                                                      | Every request (standalone mode)       | Map your session to `{ id }`                          |
| `canRead`, `canWrite`, `canManage`, `canCall`, `canForward` | Before each action                    | Custom access rules, paid tiers                       |
| `beforeMessageSend`                                         | Before save                           | Block or rewrite: length caps, profanity, PII filters |
| `afterMessageMutation`                                      | After save and publish                | Push, email, webhooks, AI replies                     |
| `beforeCallStart`, `onCallEnd`                              | Around calls                          | Limits, billing, logs                                 |
| `databaseHooks.<table>.create/update`                       | Around writes                         | Default values, audit trail                           |
| `canInstallApp`                                             | Before a Marketplace app is installed | Who may install, which apps, which scopes             |
| `beforeAiRequest`                                           | Before each model call                | Quotas, model per plan, prompt filters                |

**Client plugins** add typed actions and reactive stores (nanostores, as Better Auth uses), which become hooks for React, Vue, Svelte and Solid.

**Build your own plugin, adapter or transport**

Anything core reaches through an interface, others can supply. Each extension is a plain object passed through a small `define*` helper that type-checks it.

| Extension                  | Helper                                                          | Must pass                                                                  |
| -------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Plugin                     | `definePlugin`                                                  | Startup checks: unique id, no route, table or field clashes, typed options |
| Database adapter           | `defineAdapter`                                                 | Storage contract suite: ordering, one DM per pair, unread counts           |
| Realtime transport         | `defineTransport`                                               | Transport suite: delivery, reconnect gap-fill                              |
| Call, GIF or push provider | `defineCallProvider`, `defineGifProvider`, `definePushProvider` | That provider's suite                                                      |
| Auth resolver              | `defineResolver`                                                | Returns `{ id }` or `null`                                                 |

- **Start:** `npx @gnrllybttr/chat plugin init` scaffolds a plugin package with tests, Biome and the 100% coverage gate already wired.
- **Test kit:** `@gnrllybttr/chat-testing` ships the memory adapter, a fake transport, a fake clock and every contract suite, so a third-party adapter proves itself with one call: `runStorageContract(myAdapter)`.
- **Clashes fail fast:** startup stops with a clear error if two plugins claim the same id, route, table or field.
- **Versioning:** plugins list `@gnrllybttr/chat` as a peer dependency; plugin APIs only break in major versions.
- **Discovery:** published plugins use the npm keyword `gnrllybttrchat-plugin` and can be listed on the docs site.

**Design rules for core and plugins**

| Rule                  | In practice                                                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Single responsibility | One plugin, one feature; one adapter, one database                                                                                 |
| Open/closed           | New features arrive as plugins; core is not edited to add them                                                                     |
| Liskov substitution   | Every adapter and transport passes the same contract suite, so any can replace another                                             |
| Interface segregation | Small required interfaces, with optional capabilities (search, invites, moderation) on top                                         |
| Dependency inversion  | Core depends on interfaces (storage, transport, media, auth), never on a vendor SDK                                                |
| DRY                   | One config produces the database schema, the types and the runtime validation; one suite tests every adapter                       |
| KISS                  | Plain objects and functions: no classes, decorators or dependency-injection container. Nothing ships without a test and a real use |

## Marketplace

Anyone can list an app once, and any GnrllyBttrChat-powered app can install it, but only where that app's developer has switched the Marketplace on and an admin approves it. App A and App B each get their own install, token and data; the app never sees across them.

|                              | Plugins                                      | Marketplace apps                                     |
| ---------------------------- | -------------------------------------------- | ---------------------------------------------------- |
| Runs                         | Inside your server                           | On the app maker's server                            |
| Added by                     | The developer, in code, then a deploy        | An admin, at runtime, from what the developer allows |
| Access                       | Everything your server can do                | Only the scopes the admin approved                   |
| Shared by App A and App B    | Same npm package, installed in each codebase | Same hosted app, separate installs and tokens        |
| Listed in the Marketplace as | An npm package                               | A manifest                                           |

**How a Marketplace app works**

1. **Manifest:** JSON naming the app, its icon, the scopes it asks for (such as `messages:read`, `messages:write`, `commands`), the events it wants, its slash commands, its message-menu actions and its HTTPS endpoint.
2. **Install:** an admin approves the scopes on a consent screen. GnrllyBttrChat creates a bot user for the app and a token limited to those scopes, then sends them to the app.
3. **Events:** new messages in chats the bot was added to, slash commands and button clicks go to the app's endpoint, signed with HMAC-SHA256 over a timestamp and the body; the app rejects anything over 5 minutes old. Slack signs its app requests the same way ([Slack: verifying requests](https://docs.slack.dev/authentication/verifying-requests-from-slack/)).
4. **Actions:** the app calls your API with its token to post, reply in a thread, react or open a form.
5. **UI:** apps never run code inside your app. They send JSON blocks (text, images, buttons, menus, forms), like Slack's Block Kit, which your client draws in its own style; clicks go back to the app as events.
6. **Uninstall:** revokes the token at once, removes the bot from every chat and tells the app.

**Controls**

| Who       | Controls                                                                                                                              |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Developer | `marketplace: { enabled, allow, deny, maxScopes }`, off by default: allow every listed app, only verified publishers, or a named list |
| Developer | `canInstallApp` hook: who may install, such as the app owner only or each customer's admins                                           |
| Admin     | Installs and removes apps, approves scopes, picks which chats each bot may join                                                       |
| Users     | See an "App" badge on bot messages and can mute an app                                                                                |

- **Per customer:** if the auth resolver returns a `tenantId`, installs are per tenant (one customer org in a multi-tenant app); otherwise per app.
- **Delivery:** events queue in an outbox table and retry with backoff, so a slow app never blocks chat. On serverless, a cron job or queue runs deliveries.
- **Safety:** each install is rate-limited, every app action is logged against its install, and apps can't join encrypted chats in 1.0.

**The registry, with no server to run**

- The Marketplace is a public GitHub repo of manifests. Makers submit by pull request; CI checks the manifest and sends a signed test ping to the endpoint, then the maintainer reviews it. Raycast's Store works this way: pull requests are reviewed, then published automatically on merge ([Raycast: publish an extension](https://developers.raycast.com/basics/publish-an-extension)).
- On merge, CI publishes a signed JSON index as a static file. Each GnrllyBttrChat app fetches and caches it, and the docs site shows the same index as the browsable Marketplace.
- **Verified publisher:** the maker proves they own the endpoint's domain with a DNS record.
- **Delisting:** a removed app is flagged in the index, and apps suspend it on their next refresh.
- **Payments:** none in 1.0; paid apps bill their own customers.

**For app makers:** `@gnrllybttr/chat-app-sdk` verifies signatures, types every event and block, wraps the API, and runs a local host so an app can be built and tested without a real install.

## Repo and tooling

One Bun and Turborepo monorepo. Every check runs locally before commit and again in CI, so `main` only ever holds code that is formatted, typed, tested and buildable.

| Job                                  | Tool                        | Notes                                                                                                                                                                                              |
| ------------------------------------ | --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Runtime, installs, workspaces, tests | Bun                         | Catalogs pin each shared dependency once, in the root `package.json` ([Bun catalogs](https://bun.com/docs/pm/catalogs))                                                                            |
| Task running and caching             | Turborepo                   | Bun 1.2+ is a stable, supported package manager ([Turborepo support](https://turborepo.dev/docs/support-policy))                                                                                   |
| Lint, format, import order           | Biome v2                    | Type-aware rules such as `noFloatingPromises` without tsc; one root config that package configs extend ([Biome v2](https://biomejs.dev/blog/biome-v2/))                                            |
| Type check                           | TypeScript in `strict` mode | `tsc --noEmit` per package                                                                                                                                                                         |
| Build                                | tsdown                      | ESM plus `.d.ts`; built-in publint and attw checks catch broken package exports ([tsdown lint](https://tsdown.dev/options/lint))                                                                   |
| Dead code                            | Knip                        | Unused files, exports and dependencies across workspaces ([Knip](https://knip.dev/typescript/unused-dependencies))                                                                                 |
| Git hooks                            | Lefthook                    | pre-commit: Biome on staged files; commit-msg: commitlint ([Lefthook](https://github.com/Arkweid/lefthook/))                                                                                       |
| Commit messages                      | commitlint                  | Conventional Commits                                                                                                                                                                               |
| Versions and changelogs              | Changesets                  | CI opens the release PR                                                                                                                                                                            |
| Publishing                           | npm trusted publishing      | No stored npm token, provenance added automatically, needs npm CLI 11.5.1+ ([GitHub changelog](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/)) |
| Dependency updates                   | Renovate                    | Reads Bun's `bun.lock` ([Renovate PR](https://github.com/renovatebot/renovate/pull/33189))                                                                                                         |

- **Release flow:** merge the release PR, then CI runs `bun pm pack` per package, which swaps `workspace:` and `catalog:` for real versions, and publishes each tarball with `npm publish` ([Bun workspaces](https://bun.com/docs/install/workspaces)).
- **CI on every PR:** Biome, type check, tests at 100% coverage, contract suites on real databases, build with publint and attw, Knip, commitlint. Turborepo's cache skips unchanged packages.
- **Protection on `main`:** PRs only, every CI check required, linear history, no force pushes. Required reviews and CODEOWNERS switch on when a second maintainer joins, since GitHub won't let you approve your own PR.
- **Security:** CodeQL, secret scanning with push protection, Dependabot alerts, OpenSSF Scorecard.
- **Community files:** README, CONTRIBUTING, SECURITY, CODE_OF_CONDUCT, issue and PR templates, MIT LICENSE.

## Testing

Every change starts with a failing test, and CI blocks any merge below 100% coverage.

1. **Test first:** write the test, watch it fail, write the least code that passes, then tidy. A bug fix starts with a test that reproduces the bug.
2. **Unit tests:** `bun test` with the memory adapter, fake transport and fake clock from `@gnrllybttr/chat-testing`.
3. **Contract suites:** every adapter runs against a real database in CI: Postgres, MySQL, SQLite, MongoDB (single-node replica set), DynamoDB Local and the Firestore emulator.
4. **Integration tests:** the real HTTP handler and live stream, in-process, over both SSE and WebSocket.
5. **Calls and encryption:** calls against a LiveKit dev server; encryption with two in-process ts-mls clients.
6. **Example apps:** a smoke test each in CI, so code in the docs always runs.

- **Coverage gate:** `bunfig.toml` sets `coverageThreshold = 1`, and `bun test --coverage` exits non-zero below it. Bun enforces lines and functions but does not measure branches ([Bun coverage](https://bun.com/docs/test/coverage)).
- **Keep the gate honest:** CI keeps Bun's text reporter on, because an LCOV-only run skips the threshold check. Coverage ignores are limited to generated files.

## Packages and build order

Full 1.0 means every phase below ships before the 1.0 tag; earlier releases are 0.x. One monorepo at [GnrllyBttr/GnrllyBttrChat](https://github.com/GnrllyBttr/GnrllyBttrChat) holds `packages/*` (one per npm package), `apps/docs` and `examples/*`.

| Package                                                | Contains                                                                                                                                 |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `@gnrllybttr/chat`                                     | Core, handler, CLI, SQL adapters (Kysely, Drizzle, Prisma), memory adapter, `define*` helpers, forwarding and links, first-party plugins |
| `@gnrllybttr/chat/client`, `/react`, `/vue`, `/svelte` | Typed client and hooks                                                                                                                   |
| `@gnrllybttr/chat-testing`                             | Memory adapter, fakes (including a fake model) and contract suites, for core and third-party extensions                                  |
| `@gnrllybttr/chat-mongodb`, `-dynamodb`, `-firestore`  | Database adapters                                                                                                                        |
| `@gnrllybttr/chat-better-auth`                         | Better Auth resolver and `user` table link                                                                                               |
| `@gnrllybttr/better-auth-dynamodb`                     | Separate Better Auth adapter for DynamoDB                                                                                                |
| `@gnrllybttr/chat-ai`                                  | AI chats plugin                                                                                                                          |
| `@gnrllybttr/chat-apps`                                | Marketplace host side: installs, scopes, tokens, signed delivery, blocks, slash commands                                                 |
| `@gnrllybttr/chat-app-sdk`                             | For Marketplace app makers: signature checks, typed events and blocks, API client, local host                                            |
| `@gnrllybttr/chat-e2ee`                                | MLS encryption plugin                                                                                                                    |
| `@gnrllybttr/chat-livekit`                             | Call adapter                                                                                                                             |
| `@gnrllybttr/chat-redis`, `-ably`, ...                 | Transports for several servers or serverless                                                                                             |
| `@gnrllybttr/chat-ui`                                  | Optional unstyled components, including the blocks renderer                                                                              |

1. **Repo foundation:** Bun, Turborepo, Biome, Lefthook, CI, coverage gate and release pipeline, before any feature code.
2. **Core:** plugin API, `define*` helpers, test kit, schema system, CLI, SQL adapters, auth resolvers. DMs, groups, messages, replies, reactions, mentions, read state.
3. **Auth and databases:** Better Auth integration and other resolvers; MongoDB, Firestore and DynamoDB adapters; the Better Auth DynamoDB adapter.
4. **Realtime:** SSE and WebSocket, Redis fan-out, typing, presence, receipts, one hosted pub/sub transport.
5. **Media:** attachments, voice notes, GIF provider, link previews.
6. **Depth:** threads, mention inbox, forwarding and links, search, moderation, push.
7. **AI chats:** assistants, streaming, tools, usage limits.
8. **Calls:** LiveKit 1:1, then group, then recording.
9. **Encryption:** MLS plugin, encrypted calls, outside security review.
10. **Marketplace:** apps host package, app SDK, registry repo and index.
11. **1.0:** docs site, plugin starter, example apps, API freeze.

## Decisions

| Decision         | Choice                                                                                                                                                    |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Repo             | New repo, [GnrllyBttr/GnrllyBttrChat](https://github.com/GnrllyBttr/GnrllyBttrChat); not a Chatpack fork, though its MIT code can be borrowed with credit |
| Name             | GnrllyBttrChat                                                                                                                                            |
| Better Auth core | Not used: own handler, adapters and CLI. Better Auth stays a first-class, optional integration                                                            |
| Realtime         | SSE and WebSocket both first-class; the dev picks                                                                                                         |
| Extensibility    | Anyone can write plugins, adapters and transports; core and plugins follow SOLID, DRY and KISS                                                            |
| Testing          | TDD, 100% coverage enforced in CI, contract suites on real databases                                                                                      |
| Tooling          | Bun, Turborepo, Biome, strict TypeScript, Changesets, npm trusted publishing                                                                              |
| License          | MIT                                                                                                                                                       |
| Encryption       | In 1.0, as an opt-in plugin                                                                                                                               |
| Maintainer       | Solo for now, open source from day one                                                                                                                    |
| Scope            | Full 1.0: every phase in the build order                                                                                                                  |
| Sharing          | Forward any message or thread reply, copy links, share out; public web links opt-in                                                                       |
| AI chats         | Opt-in plugin; the developer declares each assistant                                                                                                      |
| Marketplace      | Hosted apps any GnrllyBttrChat app can install, if its developer enables it and an admin approves; registry is a GitHub repo                              |

**Still open**

- [ ] **npm names:** `@gnrllybttr/chat` and `gnrllybttrchat` were unclaimed on npm on Oct 23, 2026. Create the `@gnrllybttr` org on npm before the first publish.
- [ ] **Security review:** who reviews the encryption plugin before 1.0, and at what cost.
- [ ] **Branch coverage** (whether both sides of every `if` ran): Bun doesn't measure it. Accept lines and functions at 100%, or add a second coverage tool.
- [ ] **Marketplace repo name and reviewer:** for example `GnrllyBttr/GnrllyBttrChat-marketplace`; every listing needs a human review, which falls on the solo maintainer.
- [ ] **Marketplace in 1.0 or 1.1:** it is the largest addition and comes last in the build order, so it is the easiest to move.

## Sources

- [Chatpack on Best of JS](https://bestofjs.org/projects/chatpack)
- [Chatpack integration guide (llms.txt)](https://raw.githubusercontent.com/chddaniel/chatpack/main/llms.txt)
- [Better Auth: Plugins](https://better-auth.com/docs/concepts/plugins)
- [Better Auth: Database](https://better-auth.com/docs/concepts/database)
- [Better Auth: Create a database adapter](https://better-auth.com/docs/guides/create-a-db-adapter)
- [better-auth-firestore](https://socket.dev/npm/package/better-auth-firestore)
- [Auth.js DynamoDB adapter](https://authjs.dev/getting-started/adapters/dynamodb)
- [MongoDB change streams](https://www.mongodb.com/docs/master/changeStreams/)
- [DynamoDB TransactWriteItems](https://docs.aws.amazon.com/goto/WebAPI/dynamodb-2012-08-10/TransactWriteItems)
- [Firestore best practices](https://docs.cloud.google.com/firestore/native/docs/best-practices)
- [ts-mls README](https://cdn.jsdelivr.net/npm/ts-mls@1.6.2/README.md)
- [LiveKit README](https://github.com/livekit/livekit/blob/master/README.md)
- [PC Gamer: Google kills the Tenor API](https://www.pcgamer.com/hardware/google-kills-yet-another-api-shutting-out-all-non-google-platforms-from-favoured-gif-database/)
- [Bun: code coverage](https://bun.com/docs/test/coverage)
- [Bun: catalogs](https://bun.com/docs/pm/catalogs)
- [Bun: workspaces](https://bun.com/docs/install/workspaces)
- [Turborepo support policy](https://turborepo.dev/docs/support-policy)
- [Biome v2](https://biomejs.dev/blog/biome-v2/)
- [tsdown: lint](https://tsdown.dev/options/lint)
- [Knip](https://knip.dev/typescript/unused-dependencies)
- [Lefthook](https://github.com/Arkweid/lefthook/)
- [npm trusted publishing (GitHub changelog)](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/)
- [Renovate: bun.lock support](https://github.com/renovatebot/renovate/pull/33189)
- [MDN: navigator.share](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share)
- [AI SDK: streamText](https://ai-sdk.dev/docs/reference/ai-sdk-core/stream-text)
- [AI SDK: tools and tool calling](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling.md)
- [Slack: verifying requests](https://docs.slack.dev/authentication/verifying-requests-from-slack/)
- [Raycast: publish an extension](https://developers.raycast.com/basics/publish-an-extension)
