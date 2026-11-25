# Research: tooling

Findings behind spec §19 and §20. Gathered for the first draft (Oct 2026).

- **Bun catalogs** pin each shared dependency once, in the root `package.json` ([Bun catalogs](https://bun.com/docs/pm/catalogs)).
- **Bun workspaces:** `bun pm pack` replaces `workspace:` and `catalog:` with real versions, so tarballs can go to `npm publish` ([Bun workspaces](https://bun.com/docs/install/workspaces)).
- **Turborepo** supports Bun 1.2+ as a stable package manager ([support policy](https://turborepo.dev/docs/support-policy)).
- **Biome v2** has type-aware rules such as `noFloatingPromises` without tsc, and one root config that package configs extend ([Biome v2](https://biomejs.dev/blog/biome-v2/)).
- **tsdown** builds ESM plus `.d.ts`, with built-in publint and attw checks for broken package exports ([tsdown lint](https://tsdown.dev/options/lint)).
- **Knip** finds unused files, exports and dependencies across workspaces ([Knip](https://knip.dev/typescript/unused-dependencies)).
- **Lefthook** runs git hooks ([Lefthook](https://github.com/Arkweid/lefthook/)).
- **npm trusted publishing** needs no stored token, adds provenance, and needs npm CLI 11.5.1+ ([GitHub changelog](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/)).
- **Renovate** reads `bun.lock` ([Renovate PR](https://github.com/renovatebot/renovate/pull/33189)).
- **Bun coverage:** `coverageThreshold` in `bunfig.toml` makes `bun test --coverage` exit non-zero below it. It enforces lines and functions, not branches, and an LCOV-only reporter run skips the threshold check, so keep the text reporter on ([Bun coverage](https://bun.com/docs/test/coverage)). Branch coverage needs a second runner (ADR 13).
