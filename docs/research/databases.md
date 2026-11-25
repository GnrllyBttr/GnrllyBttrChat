# Research: databases

Findings behind spec §13. Gathered for the first draft (Oct 2026); recheck before building each adapter.

## How each NoSQL database handles the hard parts

| Database  | Message order (`seq`)                                                     | One DM per pair                                       | Live feed source       | Watch out for |
| --------- | ------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------- | ------------- |
| MongoDB   | `$inc` on the conversation doc plus insert, in one transaction            | Unique partial index on `pairKey`                     | Change streams         | Transactions and change streams need a replica set; a single-node one is enough |
| DynamoDB  | `TransactWriteItems`: bump the counter and put the message, all or nothing | Conditional put (`attribute_not_exists`) on a pair item | DynamoDB Streams     | Only key and index lookups are cheap: one index per access path, per-participant unread counters, search via an outside engine such as OpenSearch |
| Firestore | Transaction on the conversation doc's counter; messages in a subcollection | Pair key as the doc ID, created only if absent        | `onSnapshot` listeners | A single doc's update rate is limited and contends when busy; a collection with a sequentially indexed field caps at about 500 writes/s. The Admin SDK doesn't run on edge runtimes |

Sources: [MongoDB change streams](https://www.mongodb.com/docs/master/changeStreams/), [DynamoDB TransactWriteItems](https://docs.aws.amazon.com/goto/WebAPI/dynamodb-2012-08-10/TransactWriteItems), [Firestore best practices](https://docs.cloud.google.com/firestore/native/docs/best-practices).

**Open question for the adapters:** the live-feed sources above (change streams, DynamoDB Streams, `onSnapshot`) could back a `defineTransport` implementation, so a multi-server deployment needs no broker. Not required for 1.0 (ADR 5); worth a recipe.

## Better Auth on the same database

For apps that want auth and chat in one database.

| Database  | Better Auth today | Pairing |
| --------- | ----------------- | ------- |
| SQL       | Official adapters | Same database, separate tables; `chat-better-auth` adds real foreign keys to `user` |
| MongoDB   | Official adapter ([Better Auth: database](https://better-auth.com/docs/concepts/database)) | Better Auth's adapter for auth, `chat-mongodb` for chat |
| Firestore | Community [better-auth-firestore](https://socket.dev/npm/package/better-auth-firestore) (MIT) | Their adapter for auth, `chat-firestore` for chat |
| DynamoDB  | No Better Auth adapter found. Auth.js has an official [single-table adapter](https://authjs.dev/getting-started/adapters/dynamodb) | Auth lives elsewhere; a Better Auth DynamoDB adapter is out of scope (ADR 7) |

- Better Auth looks up users by email, sessions by token and accounts by provider; on DynamoDB each needs its own index, and any other filter is a full-table scan.

## Adapter contract

The generic layer (`create`, `findOne`, `findMany`, `update`, `delete`, `count`, `transaction`) mirrors Better Auth's adapter contract so its adapter authors feel at home ([Better Auth: create a DB adapter](https://better-auth.com/docs/guides/create-a-db-adapter)). It is GnrllyBttr Chat's own code.
