# 22. Schema upgrades by full regenerate

**Status:** Accepted, 2026-11-25

## Context

Each 0.x release can add tables and columns. The options were: `generate` diffs the live database, the package ships numbered migrations, or `generate` writes the full schema and developers diff it themselves.

## Decision

`generate` always writes the full schema for the current config and plugins. Developers diff and migrate with their ORM's own tools (drizzle-kit, prisma migrate). `migrate` creates the schema on an empty database. Every release with schema changes lists them in its migration note.

## Consequences

- No migration-diff engine to build or test, and no clash with custom renames.
- Developers on raw SQL without an ORM diff by hand; the migration notes must be precise.
