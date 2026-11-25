# GnrllyBttr Chat

**Chat that lives in your app, your server and your database.** Better Auth, but for chat.

> Status: design. Nothing is published yet. Read the [spec](spec.md).

Adding chat to an app usually means one of two things: pay a hosted service per monthly active user and keep your users' messages in someone else's database, or build it yourself and spend months on ordering, unread counts, reconnects and permissions.

GnrllyBttr Chat is the third option. It's an MIT-licensed TypeScript library you mount in your own server. One config gives you:

- **A schema in your database**, generated for Drizzle, Prisma or SQL, plus MongoDB, DynamoDB and Firestore. Add your own columns and they show up in your types.
- **An HTTP and realtime API**: SSE and WebSocket built in, with ordered events and gap-fill, so a dropped connection never misses a message.
- **End-to-end types**: from your config to a typed client, with React (and React Native), Vue, Svelte and Solid bindings.
- **Your auth**: first-class Better Auth, any JWT provider, or your own resolver.

```ts
import { createChat, drizzleAdapter, toNextHandler } from "@gnrllybttr/chat";
import { betterAuthResolver } from "@gnrllybttr/chat-better-auth";

export const chat = createChat({
  database: drizzleAdapter(db, { provider: "pg" }),
  auth: betterAuthResolver(auth),
});

// app/api/chat/[...all]/route.ts
export const { GET, POST } = toNextHandler(chat);
```

## For any app with chat

- **SaaS in-app messaging:** buyer and seller, coach and client, support. Optional multi-tenancy, enforced by core.
- **1:1 messengers:** DMs, read receipts, message requests, opt-in end-to-end encryption.
- **Social group chats:** groups, mentions, reactions, media, GIFs.
- **Team chat:** channels, threads, a mention inbox, and installable apps.

The base package is a small engine with 1:1 text chat. Everything else is an opt-in first-party plugin: group chat, replies, reactions, mentions, threads, media, search, push, AI assistants, voice and video calls, E2EE and apps. Together they reach parity with Slack, Discord, Teams and WhatsApp, and you install only what you use.

## Docs

- [Spec](spec.md): what 1.0 must do
- [Roadmap](docs/roadmap.md): what ships in each 0.x release
- [Decisions](docs/adr/): why it's built this way
- [Research](docs/research/): sourced findings on databases, providers and tooling

## License

MIT
