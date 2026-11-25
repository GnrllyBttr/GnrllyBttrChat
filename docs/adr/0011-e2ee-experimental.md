# 11. E2EE ships experimental in 1.0

**Status:** Accepted, 2026-11-22

## Context

`chat-e2ee` uses ts-mls, which has had no formal security audit; its authors advise an independent review before security-critical use. An audit costs time and money a solo maintainer may not have before 1.0.

## Decision

`chat-e2ee` ships in 1.0 labelled experimental and unaudited, in its docs and as a startup warning. The 1.0 tag does not wait for an audit. Core stays E2EE-ready (opaque bodies allowed, metadata readable), so the plugin can go stable without breaking changes.

## Consequences

- Apps that need audited encryption wait for the stable label.
- The audit stays an open item until it has an owner and a budget.
