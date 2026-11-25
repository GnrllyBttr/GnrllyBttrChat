# 12. Apps protocol in 1.0, central registry later

**Status:** Accepted, 2026-11-22

## Context

The Marketplace has two halves: an apps protocol inside the package (bot users, scoped tokens, signed delivery, blocks, slash commands) and a central registry (a GitHub repo of manifests, human review of every listing, a signed index, publisher verification, delisting). The registry is an ongoing service for one maintainer, not package code.

## Decision

1.0 ships the apps protocol (`chat-apps`) and the app SDK (`chat-app-sdk`). Admins install apps by manifest URL; developers allow-list by URL or domain. The central registry waits until there are apps to list and people to review them.

## Consequences

- No review burden before 1.0.
- Discovery is manual (docs, links) until the registry exists. The manifest format must not need to change when it arrives.
