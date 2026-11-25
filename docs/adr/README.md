# Architecture decision records

One file per decision: context, decision, consequences. To change a decision, add a new ADR that supersedes the old one and mark the old one `Superseded by N`.

| #  | Decision                                                            |
| -- | ------------------------------------------------------------------- |
| 1  | [Own handler and router, not Better Auth core](0001-own-handler-and-router.md) |
| 2  | [Audience: any app with chat](0002-audience.md)                     |
| 3  | [1.0 includes everything, with a core/plugin split](0003-scope.md)  |
| 4  | [Web-standard core runtime](0004-web-standard-runtime.md)           |
| 5  | [Package side vs developer side](0005-package-vs-developer-side.md) |
| 6  | [One Kysely-based SQL engine](0006-one-sql-engine.md)               |
| 7  | [Auth: Better Auth, generic JWT/JWKS, recipes](0007-auth.md)        |
| 8  | [Multi-tenancy enforced by core](0008-core-tenancy.md)              |
| 9  | [Trusted server API](0009-trusted-server-api.md)                    |
| 10 | [User data deletion and export in core](0010-user-data.md)          |
| 11 | [E2EE ships experimental in 1.0](0011-e2ee-experimental.md)         |
| 12 | [Apps protocol in 1.0, central registry later](0012-apps-protocol-now-registry-later.md) |
| 13 | [Coverage: 100% including branches](0013-coverage.md)               |
| 14 | [Naming](0014-naming.md)                                            |
| 15 | [No built-in conversation subject](0015-no-subject-field.md)        |
| 16 | [Message requests as a plugin](0016-message-requests-plugin.md)     |
| 17 | [Minimal core: engine plus 1:1 text chat](0017-minimal-core.md)     |
| 18 | [Message body: text plus typed spans](0018-message-body.md)         |
| 19 | [Idempotent send with `clientMessageId`](0019-idempotent-send.md)   |
| 20 | [Per-user change feed](0020-user-change-feed.md)                    |
| 21 | [RPC-style HTTP API](0021-rpc-style-api.md)                         |
| 22 | [Schema upgrades by full regenerate](0022-schema-upgrades.md)       |
| 23 | [Structured system messages](0023-structured-system-messages.md)    |
| 24 | [Sender identity plugin for bots and system](0024-sender-identity.md) |
| 25 | [Lockstep versions during 0.x](0025-versioning.md)                  |
| 26 | [Security and operations baseline](0026-security-and-ops.md)        |

Settled in the first draft and kept: MIT license; new repo, not a Chatpack fork (its MIT code may be borrowed with credit); TDD; Bun, Turborepo, Biome, strict TypeScript, Changesets and npm trusted publishing; solo maintainer, open source from day one.
