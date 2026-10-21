# teams-mcp

MCP server that bridges AI assistants to Microsoft Teams via the Microsoft Graph API. Uses MSAL device-code auth with a file-based token cache (`~/.teams-mcp-token-cache.json`).

## Fork

Personal fork of [floriscornel/teams-mcp](https://github.com/floriscornel/teams-mcp), published on npm as `@henriquearthur/teams-mcp`. Credit to the original author stays in `LICENSE`, `README.md`, and `package.json` `contributors`; keep it there.

- Remotes: `origin` is `henriquearthur/teams-mcp`, `upstream` is `floriscornel/teams-mcp`. Open PRs with `gh pr create --repo henriquearthur/teams-mcp`; `gh` targets the upstream parent when `--repo` is omitted.
- Sync: `git fetch upstream && git merge upstream/main` on a branch, then PR into `main`. Merge, never rebase, so `main` keeps a shared history with upstream.
- Recurring merge conflicts, and the fork's resolution:
  - `@floriscornel/teams-mcp` in new code or docs: rewrite to `@henriquearthur/teams-mcp`.
  - `.github/copilot-instructions.md` changed upstream: port the relevant change into this file, keep the Copilot file deleted.
  - Codecov steps in `ci.yml`: keep them out; the fork has no Codecov token.
  - `package.json` `version`: take whichever is higher, then follow the release steps below.
- Tags: `upstream` is configured with `tagOpt --no-tags` (`git config remote.upstream.tagOpt --no-tags`, per clone). Upstream tags pushed to `origin` would trigger the release workflow.
- Contributing back: for a change worth sending upstream, branch from `upstream/main`, keep the fork's package rename out of the diff, and open the PR with `--repo floriscornel/teams-mcp`.

## Release

`.github/workflows/release.yml` publishes to npm on a `v*` tag pushed to `henriquearthur/teams-mcp`, via npm trusted publishing (OIDC, no token). The tag must equal `package.json` `version`; the workflow fails otherwise.

1. Bump `version` in `package.json` and the `McpServer` `version` in `src/server.ts`; run `npm install --package-lock-only`. Tags `v0.9.0`–`v1.0.1` on `origin` came from upstream, so fork versions start above `1.0.1`.
2. Merge to `main`, then `git tag vX.Y.Z && git push origin vX.Y.Z`.

Trusted publishing is configured per existing package on npmjs.com, so the package's first version is published by hand (`npm login && npm publish`); after that, add `henriquearthur/teams-mcp` + `release.yml` as the trusted publisher in the package settings.

## Architecture

- **`src/index.ts`** — Dual-mode entrypoint: CLI commands (`authenticate`, `check`, `logout`) and MCP server. Routes via `process.argv`. Supports `--read-only` flag and `TEAMS_MCP_READ_ONLY` env var.
- **`src/server.ts`** — Builds the `McpServer` and registers the tool groups.
- **`src/services/graph.ts`** — Singleton `GraphService` wrapping the Microsoft Graph client. Handles token acquisition (env `AUTH_TOKEN` or MSAL silent refresh). Exports `READ_ONLY_SCOPES` and `FULL_SCOPES`; `readOnlyMode` controls which scopes are requested. All Graph API calls go through `graphService.getClient()`.
- **`src/tools/*.ts`** — Each file exports a `register*Tools(server, graphService, readOnly)` function registering MCP tools via `server.tool(name, description, zodSchema, handler)`, grouped by domain. When `readOnly` is true, write tools are skipped.
- **`src/types/graph.ts`** — Re-exports `@microsoft/microsoft-graph-types` and defines project-specific interfaces (`GraphApiResponse<T>`, `*Summary` types). Response shapes use optional properties with `| undefined` to absorb Graph API variability.
- **`src/utils/`** — Stateless helpers: `markdown.ts` (Markdown→sanitized HTML via `marked`+`DOMPurify`), `html-to-markdown.ts` (Teams HTML→Markdown via `turndown` with custom rules for `<at>`, `<attachment>`, `<systemEventMessage>`), `attachments.ts` (image hosted content), `file-upload.ts` (simple ≤4MB / resumable >4MB upload split), `users.ts` (mention processing).
- **`src/msal-cache.ts`** — File-based `ICachePlugin` for MSAL token persistence.

## Code style

- Biome is the formatter and linter (`npm run lint:fix`); CI runs `npm run ci`.
- ESM: use `.js` extensions in imports of `.ts` files, and the `node:` protocol for Node built-ins.
- TypeScript strict with `exactOptionalPropertyTypes`: optional fields are typed `T | undefined`.
- Tool handlers catch their own errors and return `{ content: [{ type: "text", text: "❌ Error: ..." }] }`.
- Cast Graph responses with `as GraphApiResponse<T>` and check `response?.value?.length` before mapping.

## Testing

- Vitest; tests live in co-located `__tests__/` directories mirroring `src/`.
- Graph HTTP calls are mocked with MSW: `src/test-utils/vitest.setup.ts` manages the server, fixtures and default handlers live in `src/test-utils/setup.ts`.
- Tool tests mock `GraphService.getClient()` to return a mock `Client` with chainable `.api().get()`/`.post()`.
- `GraphService` is a singleton: mock `@azure/msal-node` and `@microsoft/microsoft-graph-client` *before* importing the module under test. `src/services/__tests__/graph.test.ts` shows the pattern.
- Coverage threshold is 80% global (v8); `index.ts` and `test-utils/` are excluded.

## Adding a tool

1. Add or extend a `register*Tools` function in `src/tools/`, with a `zod` parameter schema.
2. Call `await graphService.getClient()` and chain `.api(endpoint)`; return `JSON.stringify(result, null, 2)` as text content.
3. Add new Graph shapes to `src/types/graph.ts`.
4. Write tests in `src/tools/__tests__/` following the mock-client pattern.
5. Register a new tool file in `src/server.ts`.

## Microsoft Graph docs

The `microsoftdocs` MCP server (`.mcp.json`) serves official Microsoft Learn docs. Use it to verify Graph endpoints, query parameters, response shapes, required permission scopes, and Teams-specific behavior (message types, mention formats, hosted content).
