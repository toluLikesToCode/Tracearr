# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Tracearr is a self-hosted monitoring platform for **Plex**, **Jellyfin**, and **Emby** media servers. It tracks live and historical playback sessions, computes analytics over them, and runs a rule engine that flags account-sharing behavior (impossible travel, simultaneous locations, device velocity, concurrent streams, geo restrictions, account inactivity) and assigns per-user trust scores.

pnpm workspace + Turborepo monorepo. Node 24 (`.tool-versions`, CI, and the Docker images all pin 24; `engines` still allows >=20). pnpm 10.28.1 via `packageManager`.

```
apps/
  server/  @tracearr/server   Fastify 5 + Drizzle + BullMQ + Socket.io  (the bulk of the code)
  web/     @tracearr/web      React 19 + Vite + Tailwind 4 + shadcn/ui
  mobile/  @tracearr/mobile   React Native / Expo Router (beta)
  e2e/     @tracearr/e2e      Playwright, drives web + server together
packages/
  shared/       Types, Zod schemas, constants, Redis key builders — imported by every app
  translations/ i18next locales + the completeness checker
  test-utils/   Factories, mocks, custom matchers, integration DB harness
```

## Commands

Everything runs from the repo root through Turbo unless noted.

```bash
pnpm install
pnpm docker:up            # TimescaleDB :5432 + Redis :6379 (docker/docker-compose.dev.yml)
cp .env.example .env      # JWT_SECRET and COOKIE_SECRET must be set or the server throws at boot
pnpm --filter @tracearr/server db:migrate
pnpm dev                  # web :5173 + server :3000 (excludes mobile)
pnpm dev:mobile           # Expo, separately
```

```bash
pnpm build         pnpm lint     pnpm lint:fix
pnpm typecheck     pnpm format   pnpm format:check
pnpm local-ci      # brings up the test DB, runs the entire PR pipeline, tears it down
```

### Tests

The server splits its suite into named vitest configs. Each has its own `vitest.<group>.config.ts` with a distinct `include` glob — a test file only runs if some config's glob matches it.

| Command                 | Covers                                                                    |
| ----------------------- | ------------------------------------------------------------------------- |
| `pnpm test:unit`        | pure functions: `utils/`, Zod schemas, mediaServer parsers, rules         |
| `pnpm test:services`    | `services/**/__tests__`, `jobs/**/__tests__` — business logic, mocked I/O |
| `pnpm test:routes`      | `routes/__tests__`, `routes/stats/__tests__` — handlers with mocked DB    |
| `pnpm test:security`    | every `*.security.test.ts` — authn/authz behavior, no coverage gate       |
| `pnpm test:integration` | `apps/server/test/integration/*.integration.test.ts` against a real DB    |
| `pnpm test:e2e`         | Playwright; boots real server + web                                       |
| `pnpm test`             | everything under `src/**/*.test.ts` except integration and stress         |
| `pnpm test:coverage`    | the gated run — thresholds live in `vitest.config.ts`                     |

Running one file or one test — extra args pass straight through to vitest:

```bash
pnpm --filter @tracearr/server test:unit src/utils/__tests__/crypto.test.ts
pnpm --filter @tracearr/server test:services -t "impossible travel"
pnpm --filter @tracearr/server test:watch
```

Integration tests need `docker compose -f docker/docker-compose.test.yml up -d --wait` (TimescaleDB :5433, Redis :6380) and honor `TEST_DATABASE_URL` / `TEST_REDIS_URL`. They run single-fork and non-parallel because they share one database.

E2E needs the _dev_ stack up (:5432/:6379) plus `pnpm --filter @tracearr/e2e exec playwright install chromium` once. Playwright starts both servers itself and reuses them if already running.

### Database

```bash
pnpm --filter @tracearr/server db:generate   # edit src/db/schema.ts first, then generate
pnpm --filter @tracearr/server db:migrate
pnpm --filter @tracearr/server db:studio
pnpm --filter @tracearr/server reset-password
```

Migrations are drizzle-kit generated SQL in `apps/server/src/db/migrations/` and are applied automatically on server startup. Never hand-edit an existing migration or the `meta/` journal — change `schema.ts` and generate a new one.

### Translations

`pnpm translations:check` runs in CI and in the pre-push hook. Add new strings to `packages/translations/src/locales/en/` (the base language); `pnpm translations:fix` backfills the other ~30 locales with English defaults, which Crowdin later replaces.

## Architecture

### Server startup is two-phase and reversible

`apps/server/src/index.ts` is the whole lifecycle and worth reading before changing anything about boot order.

1. **`buildApp()`** always succeeds. Plugins, routes, and `/health` register even when Postgres and Redis are unreachable. An `onRequest` hook gates every `/api/*` route behind `isMaintenance()` and returns 503; non-API paths still serve the SPA so the frontend can render its maintenance page.
2. **`initializeServices()`** runs only once DB _and_ Redis both probe healthy: migrations, prepared statements, TimescaleDB setup, GeoIP/GeoASN, the V2 rules wiring, every BullMQ queue, and the poller/SSE managers.
3. **`initializePostListen()`** attaches Socket.io to Fastify's HTTP server, subscribes to the Redis pub/sub channel, and starts the poller and Plex SSE connections.

`serverState.ts` holds the `starting | maintenance | ready` singleton plus DB/Redis health flags. Losing Redis (a `close` event) or failing the 10s DB health poll flips the server back to maintenance, which tears down queues and timers to stop reconnect log floods; a recovery loop probes every 10s and re-runs phases 2 and 3. **Any new interval, queue, or long-lived connection must be registered in both the maintenance teardown and the re-init path**, or it will leak or fail to come back.

### Session ingestion: poller and SSE

Two independent paths write sessions, and they must not double-write:

- **Poller** (`src/jobs/poller/`) — `setInterval`, not BullMQ. Split into `processor` (orchestration), `sessionMapper`, `stateTracker` (pause accumulation, 85% watch-completion, resume grouping), `sessionLifecycle`, `pendingConfirmation`, `violations`, `database`. Handles all three server types.
- **SSE** (`src/services/sseManager.ts` + `src/jobs/sseProcessor.ts`) — Plex only, since Plex is the only backend that pushes. An SSE event carries minimal data, so the processor refetches full session details from Plex and then reuses the poller's lifecycle functions.

Shared session writes go through `createSessionWithRulesAtomic` / `stopSessionAtomic` / `confirmAndPersistSession`, exported from `jobs/poller/index.ts`, and are serialized by `cacheService.withSessionCreateLock()` (Redis) so SSE and the poller can't race on the same `serverId:sessionKey`.

### Media server abstraction

`services/mediaServer/` hides the three backends behind `createMediaServerClient({ type, url, token })` returning `IMediaServerClient`. Per-vendor `client.ts` (HTTP) is kept separate from `parser.ts` (pure response → domain mapping) so parsers are unit-testable with no network. Jellyfin and Emby share most logic via `shared/jellyfinEmbyParser.ts` and `shared/baseMediaServerClient.ts`. Watch-history support is optional and detected with the `supportsWatchHistory()` type guard. **New behavior belongs in a parser, not a client**, unless it genuinely needs I/O.

### Rules: two generations coexist

- **V1** (`services/rules.ts`) — the six fixed rule types with typed params from `RULE_DEFAULTS` in `@tracearr/shared`.
- **V2** (`services/rules/`) — a generic condition/action engine: `engine.ts` walks condition groups, `evaluators/` resolves fields, `executors/` runs actions (with `targeting.ts` deciding who an action applies to), `comparisons.ts` holds operators. `v2Integration.ts` injects real dependencies into the executors and migrates V1 rows on startup; `migration.ts` does the translation.

The engine re-evaluates selectively: `hasTranscodeConditions()` and `hasPauseConditions()` gate which rules get re-run on transcode transitions and on each poll of a paused session. Adding a condition field that changes mid-session means adding it to the matching set in `engine.ts`, or the rule will only ever fire at session start.

### TimescaleDB

`sessions` is a hypertable with continuous aggregates defined in `src/db/timescale.ts`. **`AGGREGATE_SCHEMA_VERSION` at the top of that file must be incremented whenever an aggregate's SQL or shape changes** — that constant is what triggers a rebuild on the next startup. Each aggregate's comment records which routes consume it; unused ones have been deleted before, so keep that mapping honest.

The app degrades gracefully to plain PostgreSQL if the extension is missing, just slower.

### Media type filtering

`src/constants/mediaTypes.ts` is the single source of truth. `PRIMARY_MEDIA_TYPES` (`movie`, `episode`) drives dashboard/plays/user stats; `EXCLUDED_MEDIA_TYPES` (live TV, music tracks, photos, trailers, unknown) are tracked but kept out of primary stats and rule evaluation. The SQL literals are derived from the TS arrays — change the arrays, not the SQL.

### Background jobs

BullMQ queues, all initialized in `index.ts` with the same `init* / start*Worker / shutdown*` shape: notification, import, maintenance, librarySync, versionCheck, inactivityCheck, backup. Import and maintenance additionally contend for `jobs/heavyOpsLock.ts`, a Redis lock that prevents concurrent heavy TimescaleDB work (and exposes what a blocked job is waiting on, for the UI).

### Real-time fan-out

Services publish to a Redis pub/sub channel (`REDIS_KEYS.PUBSUB_EVENTS`); a dedicated subscriber in `initializePostListen` translates each `WS_EVENTS.*` message into a Socket.io broadcast. This indirection is what lets background workers emit to clients without holding a socket. New event types need the constant in `@tracearr/shared`, a `case` in that switch, and a client handler in `apps/web/src/hooks/useSocket.tsx`.

### Redis keys

Never build a key by hand. `REDIS_KEYS` in `packages/shared/src/constants.ts` uses getters and factory functions so the optional `REDIS_PREFIX` env var (for shared Redis instances) applies uniformly. The prefix is installed once via `setRedisPrefix()` in the Redis plugin.

### Settings

Runtime configuration lives in a key-value `settings` table, not env vars. Go through `services/settings.ts` — it owns `PUBLIC_DEFAULTS` (exposed by `GET /settings`) and `INTERNAL_DEFAULTS` (not exposed), and applies defaults for absent keys. Adding a setting means extending those objects, not querying the table directly.

### Auth

`plugins/auth.ts` decorates Fastify with four guards: `authenticate` (JWT via cookie or bearer), `requireOwner`, `requireMobile`, and `authenticatePublicApi` (read-only `trr_pub_` API keys, surfaced through Swagger UI at `/api-docs`). A `jwtRevokedBefore` setting acts as a global token kill-switch — tokens issued before that timestamp are rejected, which is how restore invalidates old sessions.

### Frontend

- `src/lib/api.ts` is the only place that talks HTTP. `src/hooks/queries/*` wraps it in React Query hooks; components consume the hooks, never `api` directly. Query keys are hierarchical arrays (`['sessions', 'active', key]`) so mutations can invalidate by prefix.
- `BASE_PATH` (subpath deployments) is pervasive: the server strips it in `rewriteUrl` and injects a `<base>` tag into `index.html`; Vite proxies through it in dev; the client reads it from `src/lib/basePath.ts`. **`apps/server/src/routes/__tests__/` contains a static-analysis test that fails CI on hardcoded `fetch('/...')` calls in web source** — always route requests through the API client.
- Server state via React Query, toasts via `sonner`, forms via react-hook-form + Zod, charts via Highcharts, maps via Leaflet.
- User-facing strings go through `react-i18next` (`useTranslation('namespace')`), not literals.
- Design tokens are documented in `docs/style-guide.md`; the accent color derives from a single `--accent-hue` CSS variable in `apps/web/src/styles/globals.css`.

## Conventions

**TypeScript.** `strict`, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, `isolatedModules` (see `tsconfig.base.json`). Server and packages are ESM with `moduleResolution: NodeNext`, so **relative imports need the `.js` extension** (`./db/client.js`) even though the source is `.ts`. Web uses `moduleResolution: bundler` and the `@/` alias, no extensions. Type-only imports must use inline `type` — ESLint enforces `consistent-type-imports`.

**Lint.** `typescript-eslint` strict + stylistic type-checked. The `no-unsafe-*` family and `no-explicit-any` are warnings, not errors, and are relaxed entirely in test files — don't "fix" pre-existing warnings as drive-by work.

**Naming.** PascalCase for components (`UserProfile.tsx`), camelCase for everything else (`sessionService.ts`). Server tests sit in `__tests__/` next to the code; security tests are named `*.security.test.ts`.

**Commits.** Plain-language, imperative, no conventional-commit prefixes — `add session termination endpoint`, not `feat(sessions): ...`. See `CONTRIBUTING.md`.

**Hooks.** `pre-commit` runs prettier on staged files. `pre-push` runs `typecheck`, `test:unit`, `lint`, and `translations:check` — expect a push to take a minute.

**Tests.** `packages/test-utils` provides factories (`buildSession`, `buildUser`, …), an ioredis-mock, media-server mocks, custom matchers, and the integration DB harness; it's imported via subpath exports (`@tracearr/test-utils/factories`). Both vitest and the test suite resolve `@tracearr/shared` to _source_ but `@tracearr/test-utils` to `dist`, so **build test-utils before running tests after changing it** (`pnpm turbo run build --filter=@tracearr/test-utils`).

**PRs.** `.github/pull_request_template.md` includes an AI-disclosure checkbox; `CONTRIBUTING.md` asks that significant AI assistance be noted. Features should be discussed in an issue/Discussion before implementation; bug fixes can go straight to a PR.
