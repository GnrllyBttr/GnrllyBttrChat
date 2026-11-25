# 18. Message body: text plus typed spans

**Status:** Accepted, 2026-11-25

## Context

The body format was undefined, and every client, search, push preview, AI context and E2EE depends on it. The options were a Markdown string, text plus typed spans, or a rich document tree (ProseMirror or Lexical style).

## Decision

A body is `text` plus `spans`: typed ranges over the text. Core span types are `link`, `bold`, `italic`, `strike`, `code` and `codeBlock`; plugins add theirs (`mention`, `emoji`) with a payload schema. Offsets are UTF-16 code units and are validated. Client helpers convert Markdown in and out.

## Consequences

- Search, push and AI read `text` directly, with no parsing.
- Mentions survive renames, because spans hold user ids.
- In encrypted conversations `text` is ciphertext and `spans` is empty.
- No nested blocks (lists, tables); apps that need them use a plugin message kind.
