# 8. Multi-tenancy enforced by core

**Status:** Accepted, 2026-11-22

## Context

The first draft used `tenantId` only to scope Marketplace installs. In multi-tenant SaaS every conversation and query needs it, or one customer org can read another's data. Leaving isolation to `canRead` and `canWrite` hooks makes it each developer's job, and easy to get wrong.

## Decision

When the auth resolver returns `tenantId`, core stamps it on every row and filters every read, count, search and subscription by it. Plugin tables are tenant-scoped by default. A storage contract test proves no read crosses tenants. The column is always generated, nullable.

## Consequences

- Every adapter and plugin carries tenant filters; the contract suite catches leaks.
- Apps without tenants pay only an unused nullable column, and can turn tenancy on later with a data migration rather than a schema change.
