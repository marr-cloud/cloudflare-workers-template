# AGENTS.md

Instructions for AI coding agents working in this repository. It is a framework-agnostic starter
template for a single Cloudflare Worker (TypeScript, Vitest in `workerd`, oxlint + oxfmt, pnpm).

## Cloudflare docs first

STOP. Your knowledge of Cloudflare Workers APIs and limits may be outdated. Always retrieve current
documentation before any Workers, KV, R2, D1, Durable Objects, Queues, Vectorize, AI, or Agents SDK task.

- Docs: https://developers.cloudflare.com/workers/
- MCP: `https://docs.mcp.cloudflare.com/mcp`
- Limits and quotas: each product's `/platform/limits/` page, e.g. `/workers/platform/limits/`
- Product docs: `/kv/` · `/r2/` · `/d1/` · `/durable-objects/` · `/queues/` · `/vectorize/` · `/workers-ai/` · `/agents/`
- Node.js compatibility: https://developers.cloudflare.com/workers/runtime-apis/nodejs/
- Errors: https://developers.cloudflare.com/workers/observability/errors/ — **1102** means CPU/memory
  exceeded; check the limits page.
- If the app uses Durable Objects or Workflows, follow their best practices:
  [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/) ·
  [Rules of Workflows](https://developers.cloudflare.com/workflows/build/rules-of-workflows/)

The same applies to any library you add (Hono, Zod, …): read its current docs instead of assuming APIs.

## Project map

| Path                        | Purpose                                                                  |
| --------------------------- | ------------------------------------------------------------------------ |
| `src/index.ts`              | Worker entry point: `export default { fetch } satisfies ExportedHandler` |
| `test/index.spec.ts`        | Tests, run inside `workerd` by `@cloudflare/vitest-plugin`               |
| `test/env.d.ts`             | Types `env` from `cloudflare:workers` as `CloudflareBindings`            |
| `wrangler.jsonc`            | Worker config: bindings, vars, compatibility date/flags, observability   |
| `worker-configuration.d.ts` | **Generated** by `pnpm cf-typegen`. Never edit, never format             |
| `.oxlintrc.json`            | Lint config (plugins, categories, type-aware rules)                      |
| `.oxfmtrc.jsonc`            | Formatter config and ignore patterns                                     |
| `vitest.config.mts`         | Vitest + Cloudflare plugin, reads `wrangler.jsonc`                       |
| `.dev.vars.example`         | Documents local secrets; copy to `.dev.vars` (gitignored)                |
| `.github/workflows/`        | `ci.yml` (push/PR), `deploy.yml` (manual), `dependabot-typegen.yml`      |

## Commands

Use `pnpm` (never npm/yarn). Run local binaries with `pnpm exec <bin>`.

| Command                  | Purpose                                                  |
| ------------------------ | -------------------------------------------------------- |
| `pnpm dev`               | Local dev server (`wrangler dev`, http://localhost:8787) |
| `pnpm test`              | Run tests once                                           |
| `pnpm test:watch`        | Tests in watch mode                                      |
| `pnpm typecheck`         | `tsc` over `src/` and `test/`                            |
| `pnpm lint` / `lint:fix` | oxlint (type-aware) / with autofix                       |
| `pnpm format`            | Format everything with oxfmt                             |
| `pnpm format:check`      | Check formatting                                         |
| `pnpm cf-typegen`        | Regenerate `worker-configuration.d.ts`                   |
| `pnpm cf-typegen:check`  | Fail if the generated types are stale                    |
| `pnpm check`             | All gates, same order as CI                              |
| `pnpm deploy`            | `wrangler deploy` — only when the user asks              |

## Quality gates

Before saying a change is done, run `pnpm check` and read the output. It runs, in order:
`lint` → `format:check` → `cf-typegen:check` → `typecheck` → `test`. CI runs the same steps.

- Lint or format failures: run `pnpm lint:fix` / `pnpm format`, then re-check. Do not disable rules
  to get green; if a rule is genuinely wrong for the case, use a scoped
  `// oxlint-disable-next-line <rule> -- <reason>`.
- `cf-typegen:check` failure: run `pnpm cf-typegen` and commit the regenerated file. Wrangler's
  error message says "Run `wrangler types`" — ignore that wording: bare `wrangler types` generates
  an `Env` interface instead of `CloudflareBindings` and breaks every type reference.
  On Dependabot PRs, `dependabot-typegen.yml` does this automatically.

## Conventions

- **Formatting**: tabs, 140 columns, double quotes, semicolons, sorted imports — oxfmt owns it.
- **Types**: `strict` + `noUncheckedIndexedAccess` + `verbatimModuleSyntax` (use `import type`).
  The env type is the generated `CloudflareBindings`; never hand-write an `Env` interface.
- **Promises**: every promise is awaited or passed to `ctx.waitUntil()` for work that should outlive
  the response. A floating promise is cancelled when the response ends (and fails lint).
- **Logging**: structured JSON, `console.log(JSON.stringify({ event, ... }))`, so logs stay queryable
  in Workers Logs. Never log secrets or full request bodies.
- **Config**: non-sensitive values in `vars` in `wrangler.jsonc`. Secrets go in `.dev.vars` locally
  and `pnpm exec wrangler secret put NAME` in production — never in source or `wrangler.jsonc`.
- **Compatibility date**: bump it deliberately, then run `pnpm cf-typegen`.
- **Dependencies**: dev tooling only by default. Add runtime deps only when needed and check they
  run on `workerd` (with `nodejs_compat`).

## Adding a binding

1. Add it to `wrangler.jsonc` (commented examples for KV, R2, D1, assets, services are in the file).
2. Run `pnpm cf-typegen`.
3. Use it as `env.MY_BINDING`; tests get it from `cloudflare:workers` automatically.

Prefer bindings over calling the Cloudflare REST API from inside the Worker.

## Testing

Tests run in the real `workerd` runtime with bindings from `wrangler.jsonc`. Two patterns exist in
`test/index.spec.ts`; follow them:

- **Unit style**: build a request, call `worker.fetch(request, env, ctx)` with
  `createExecutionContext()` from `cloudflare:test`, then `await waitOnExecutionContext(ctx)` before
  asserting (this flushes `ctx.waitUntil` work).
- **Integration style**: `exports.default.fetch("https://example.com/...")` from `cloudflare:workers`
  exercises the Worker as deployed.

Add or update a test with every behavior change. Assert status, headers and body.

## Frameworks

The template is framework-agnostic. If the user wants Hono (or another router), follow the
"Using a framework (Hono)" section of `README.md`: install it, type the app with
`{ Bindings: CloudflareBindings }`, `export default app`, and use `app.request(path, init, env)` in
unit tests.

## Don't

- Edit or format `worker-configuration.d.ts` by hand.
- Run bare `wrangler types` — always `pnpm cf-typegen` (it passes `--env-interface CloudflareBindings`).
- Add Prettier, ESLint or another formatter/linter — the oxc toolchain covers both.
- Commit `.dev.vars`, `.env*`, tokens or account IDs.
- Run `wrangler deploy`, `wrangler secret put` or anything touching a live account without the
  user asking for it.
- Leave planning or scratch documents in the repository.
