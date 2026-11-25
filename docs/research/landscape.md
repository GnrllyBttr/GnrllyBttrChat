# Research: landscape and providers

Findings behind the README and spec §10, §12 and §15. Gathered for the first draft (Oct 2026).

## Chatpack

The closest existing project: an MIT, in-app chat backend ([Best of JS](https://bestofjs.org/projects/chatpack), [integration guide](https://raw.githubusercontent.com/chddaniel/chatpack/main/llms.txt)). Gaps found in its guide:

| Chatpack                                    | GnrllyBttr Chat                                              |
| ------------------------------------------- | ------------------------------------------------------------ |
| Fixed schema, no generate step              | Schema from config and plugins; `generate` and `migrate`     |
| Custom data only in a `metadata` JSON field | Typed `additionalFields` that flow into server and client types |
| Falls back to polling on serverless, losing typing, presence and receipts | Event contract plus pluggable transports (ADR 5) |
| Stores mentions, but no inbox or notifications | Mention inbox, badges, `mention.created`                  |
| No threads, push, calls, encryption, sharing, AI or apps | All in 1.0 (ADR 3)                              |

Both are resolver-based for auth; GnrllyBttr Chat adds a first-class Better Auth integration and a JWT/JWKS resolver (ADR 7).

**To do before the docs site:** extend this into a comparison with Stream, Sendbird, and rolling your own.

## Providers

- **GIFs:** Google shut down the Tenor API on June 30, 2026; Discord, WhatsApp and Bluesky moved to Klipy ([PC Gamer](https://www.pcgamer.com/hardware/google-kills-yet-another-api-shutting-out-all-non-google-platforms-from-favoured-gif-database/)). Reference adapter: Klipy. No Tenor adapter.
- **Calls:** LiveKit is Apache 2.0, self-hosts as a single binary or Docker (or LiveKit Cloud), authenticates joins with JWT access tokens, sends webhooks, includes TURN relays, records through Egress, and supports end-to-end encrypted media ([LiveKit README](https://github.com/livekit/livekit/blob/master/README.md)). A media server is used even for 1:1, because 1:1 still needs relays to get through firewalls and one path is simpler.
- **E2EE:** MLS (RFC 9420) via [ts-mls](https://cdn.jsdelivr.net/npm/ts-mls@1.6.2/README.md), MIT. No formal audit; its authors advise an independent review before security-critical use.
- **AI:** the AI SDK puts OpenAI, Anthropic and other providers behind one interface ([streamText](https://ai-sdk.dev/docs/reference/ai-sdk-core/stream-text), [tools](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling.md)).
- **Apps:** request signing follows Slack's scheme, HMAC-SHA256 over a timestamp and body ([Slack: verifying requests](https://docs.slack.dev/authentication/verifying-requests-from-slack/)). A future registry could follow Raycast's model: PRs reviewed, then published on merge ([Raycast](https://developers.raycast.com/basics/publish-an-extension)).
- **Share sheet:** [MDN: navigator.share](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share); not in every browser.
