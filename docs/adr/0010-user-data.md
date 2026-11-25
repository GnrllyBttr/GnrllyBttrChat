# 10. User data deletion and export in core

**Status:** Accepted, 2026-11-22

## Context

GDPR and app-store rules (Apple requires in-app account deletion) mean apps must delete and export a user's data. Chat data spans core and plugin tables in the developer's database, so only the package knows where it all is.

## Decision

Core ships `chat.api.users.delete(userId)`, which anonymizes or hard-deletes messages per config, and `chat.api.users.export(userId)`. Every plugin with user-linked tables must implement `onUserDelete` and contribute to export; `definePlugin` fails at startup otherwise.

## Consequences

- Plugin authors carry one small extra requirement.
- Anonymized messages keep threads and replies readable after a user leaves.
