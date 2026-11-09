# @sudobility/shapeshyft_service

Shared backend for the `shapeshyft_api` and `shaperouter_api` products: Drizzle
table factories, idempotent DDL, Hono routers, and auth / rate-limit
middleware for ShapeShyft-style structured-output LLM APIs. It is published as an
ESM library (`dist/`), not a standalone server — each product wires it up and
runs it.

Stateless LLM work (provider adapters, prompt building, media, the provider
catalog) lives in the sibling package `@sudobility/shapeshyft_engine`; this
package owns the database schema, HTTP routes, and the request lifecycle.

## What it provides

- **Table factories** (`createServiceTables`) — the shared schema (users,
  entities, projects, endpoints, API keys, usage analytics, rate-limit
  counters…) built against a caller-supplied `pgSchema`, so each product keeps
  its own PostgreSQL schema. Everything except `llm_api_keys`.
- **Idempotent DDL** (`initServiceTables`) for bootstrapping those tables.
- **Hono routers** assembled by `createShapeshyftService(...).buildRoutes()`
  into the `/api/v1` route tree (public `ai`/`providers` routes plus
  authenticated admin routes for entities, projects, endpoints, analytics,
  settings, rate limits, invitations, storage, users, and API keys).
- **Middleware** — Firebase auth / API-key auth, subscription lookups, and
  rate limiting.
- **The provider-credential seam** — the app supplies a
  `ProviderCredentialResolver`; the service never learns where a provider API
  key is stored (see below).
- **Process-bootstrap factories** (`src/server/`) — env reader, lazy database,
  Firebase auth, invitation email (Resend), and the Bun HTTP server — so the
  product shells don't duplicate them.

## Install

This package and its `@sudobility/*` dependencies are published to **GitHub
Packages**. `.npmrc` routes the `@sudobility` scope to
`https://npm.pkg.github.com`; installing requires a GitHub token with
`read:packages` in your personal `~/.npmrc` (see the `GH_PACKAGES_READ_TOKEN`
note in CI).

```sh
bun install
```

Bun is the package manager and script runner (`bunfig.toml` keeps this repo
standalone, outside any parent workspace). The `@sudobility/*` packages and
`drizzle-orm`, `firebase-admin`, `hono`, and `postgres` are **peer
dependencies** — the consuming app provides them.

## Scripts

| Command | What it does |
| --- | --- |
| `bun run build` | Compile ESM + types to `dist/` (`tsc -p tsconfig.esm.json`). |
| `bun run typecheck` | Type-check only, no emit. |
| `bun run lint` / `bun run lint:fix` | ESLint over `src/`. |
| `bun run format` | Prettier over `src/` and `tests/`. |
| `bun run test:unit` | Run the unit suite (Vitest, no database). |
| `bun run test:watch` | Vitest in watch mode. |
| `bun run test:db` | Run the DB-backed suite (requires `TEST_DATABASE_URL`). |
| `bun run verify` | `typecheck` + `lint` + `test:unit` + `build`. |
| `bun run clean` | Remove `dist/`. |

> There is **no** `test` script — unit tests are `bun run test:unit`.

### Database tests

The DB suite talks to a real PostgreSQL instance and refuses anything that
isn't localhost (guarded by `@sudobility/test-db-guard`):

```sh
TEST_DATABASE_URL=postgresql://localhost:5432/shapeshyft_test bun run test:db
```

Unit and DB suites use separate Vitest configs (`vitest.config.ts` excludes
`*.db.test.ts`; `vitest.db.config.ts` runs only those, with file parallelism
disabled since they share one database).

## Usage

```ts
const service = createShapeshyftService({
  db,
  tables,
  keyPrefixes: { user: "shyft_", entity: "shyftent" },
  encryption,
  auth,
  email,
  credentials, // ProviderCredentialResolver: where provider API keys come from
  getPeerAddress, // e.g. c => getConnInfo(c).remote.address, from hono/bun
});

app.route("/api/v1", service.buildRoutes({ mountAdmin }));
```

### The credential seam

The service never knows where a provider API key lives. The app passes a
`ProviderCredentialResolver`:

- `bindEndpoint` — endpoint create/update; returns the `provider` and
  `llmKeyId` to persist, or a `{ status, message }` failure.
- `resolve` — invoke and prompt-preview time; returns `provider`, `apiKey`,
  `endpointUrl`, `timeoutMs`, or a failure.

Call-time overrides `llm_provider` / `llm_model` are reserved input fields
(stripped like `context`). An invalid `llm_provider` is a 400; an omitted
`llm_model` falls back to the catalog `defaultModel`. See `CLAUDE.md` for the
full lifecycle, the `/prompt` request-preview contract, client-IP allowlist
rules, and conventions.

## License

BUSL-1.1 © Sudobility
