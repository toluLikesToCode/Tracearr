# 🚀 Tracearr Developer Onboarding Guide

Welcome to the team. Tracearr is a **streaming access manager** — think of it as a security and
analytics layer that sits on top of Plex, Jellyfin, and Emby. Users connect their media servers,
and Tracearr monitors sessions in real-time, enforces access rules (impossible travel, concurrent
streams, geo-blocks, etc.), and generates rich analytics. It's a startup — you'll touch real
features fast, and your work ships to real users.

---

## Part 1 — Environment Setup

### Prerequisites

First, verify your machine has the right tools. Tracearr uses `.tool-versions` to pin Node:

```
nodejs 24.11.0
```

**Install these if you don't have them:**

| Tool               | Version         | Install                                                                   |
| ------------------ | --------------- | ------------------------------------------------------------------------- |
| Node.js            | ≥ 20 (pin 24.x) | [nvm](https://github.com/nvm-sh/nvm) or [fnm](https://github.com/Schniz/fnm) recommended |
| pnpm               | ≥ 9 (10.28.x)   | `npm i -g pnpm`                                                           |
| Docker + Compose   | Latest          | [Docker Desktop](https://www.docker.com/products/docker-desktop/)        |
| Git                | Latest          | OS package manager                                                        |

> **Why pnpm?** The monorepo uses pnpm workspaces. Using `npm` or `yarn` will break workspace
> resolution. Always use `pnpm`.

---

### Step 1 — Clone & Install

```bash
git clone https://github.com/toluLikesToCode/Tracearr.git
cd Tracearr
pnpm install
```

This installs dependencies across all workspaces (`apps/*` and `packages/*`) at once. Turborepo
handles the dependency graph.

---

### Step 2 — Environment Variables

```bash
cp .env.example .env
```

Open `.env` and review — the defaults work for local development. The important fields:

```bash
DATABASE_URL=postgresql://tracearr:tracearr@localhost:5432/tracearr
REDIS_URL=redis://localhost:6379
PORT=3000
NODE_ENV=development
JWT_SECRET=your-secret-key-change-in-production    # any string locally
COOKIE_SECRET=your-cookie-secret-change-in-production
CORS_ORIGIN=http://localhost:5173                   # Vite dev server
```

> **Security note:** The `JWT_SECRET` and `COOKIE_SECRET` are dummy values for local dev only.
> In production these must be generated with `openssl rand -hex 32`.

---

### Step 3 — Start the Databases

Tracearr uses **TimescaleDB** (a PostgreSQL extension for time-series data) and **Redis**.
Docker Compose handles both:

```bash
pnpm docker:up
```

This builds a custom TimescaleDB image and starts both services. When they're healthy (green in
Docker Desktop), you're ready for migrations.

To shut everything down:

```bash
pnpm docker:down
```

---

### Step 4 — Run Database Migrations

```bash
pnpm --filter @tracearr/server db:migrate
```

This runs all pending Drizzle ORM migrations from `apps/server/src/db/migrations/`. The schema
is the source of truth — never edit the database directly.

---

### Step 5 — Start the Dev Server

```bash
pnpm dev
```

This command, powered by Turborepo, starts **both** the backend server and the frontend
simultaneously in watch mode:

- **Backend (Fastify):** `http://localhost:3000` — hot-reloads via `tsx watch`
- **Frontend (Vite + React):** `http://localhost:5173` — HMR

Go to `http://localhost:5173` and you'll be redirected to `/setup` to configure your first media
server. You don't need a real Plex or Jellyfin server to explore the codebase, but you'll need
one to test live session features.

> **Tip:** `pnpm dev` excludes mobile. To include it: `pnpm dev:all`. Mobile is a separate
> Expo app requiring `pnpm dev:mobile`.

---

## Part 2 — Workflow Setup

### IDE Setup

The repo ships with VS Code config (`.vscode/`). If you use VS Code, you'll get recommended
extensions and settings automatically.

**Recommended extensions:**

- ESLint
- Prettier
- Tailwind CSS IntelliSense
- Drizzle Kit (for schema visualization)

### Git Hooks (Husky + lint-staged)

When you run `pnpm install`, Husky sets up a pre-commit hook automatically. **Before every
commit**, `lint-staged` runs Prettier on all staged `.ts/.tsx/.js/.json/.md` files. This means:

- You don't need to manually format before committing
- But you _should_ still run `pnpm lint` and `pnpm typecheck` before pushing (CI will catch it
  otherwise)

### Everyday Development Commands

```bash
# Type-check everything
pnpm typecheck

# Lint everything
pnpm lint
pnpm lint:fix      # Auto-fix where possible

# Format everything
pnpm format

# Run fast unit tests (run these constantly)
pnpm test:unit

# Run all tests before a PR
pnpm lint && pnpm typecheck && pnpm test:unit && pnpm test:services && pnpm test:routes

# DB operations
pnpm --filter @tracearr/server db:generate   # After schema changes
pnpm --filter @tracearr/server db:migrate    # Apply migrations
pnpm --filter @tracearr/server db:studio     # Visual DB browser
```

### PR Workflow

1. Branch off `main` — use descriptive names (`fix-session-dedup`, `add-geo-restriction-rule`)
2. Write tests alongside your code — the project has a strong test culture
3. Before opening a PR: `pnpm lint && pnpm typecheck && pnpm test:unit`
4. PR description: explain _what_ changed and _why_, add screenshots for UI work
5. Commit messages are plain English — `Add impossible travel grace period config`, not
   `feat(rules): add grace period`

---

## Part 3 — Codebase Architecture

### Monorepo Structure

```
Tracearr/
├── apps/
│   ├── server/          # Fastify API server (Node.js backend)
│   ├── web/             # React 19 SPA (admin dashboard)
│   ├── mobile/          # React Native (Expo) mobile app
│   └── e2e/             # End-to-end tests
├── packages/
│   ├── shared/          # Types, Zod schemas, constants — shared by all apps
│   ├── translations/    # i18n strings (i18next)
│   └── test-utils/      # Test factories and mocks
├── docker/              # Docker Compose files & Dockerfiles
├── turbo.json           # Turborepo pipeline config
└── pnpm-workspace.yaml  # Workspace definition
```

**Turborepo** is the build orchestrator. It understands the dependency graph between packages,
caches builds, and runs tasks in parallel. When you run `pnpm build`, Turborepo builds
`@tracearr/shared` first (since `apps/server` and `apps/web` depend on it), then builds them in
parallel.

---

### The Server (`apps/server`)

The backend is a **Fastify 5** application written in TypeScript ESM.

```
src/
├── index.ts          # App bootstrap — registers plugins, routes, starts server
├── db/
│   ├── schema.ts     # Drizzle ORM table definitions (THE source of truth)
│   ├── client.ts     # DB connection pool
│   ├── prepared.ts   # Prepared statements for hot paths
│   └── migrations/   # Generated SQL migrations (never edit manually)
├── routes/           # HTTP route handlers (one file per domain)
│   ├── sessions.ts
│   ├── servers.ts
│   ├── rules.ts
│   ├── users/
│   └── ...
├── services/         # Business logic (pure functions or classes)
│   ├── rules.ts      # Rule evaluation engine (RuleEngine class)
│   ├── mediaServer/  # Plex/Jellyfin/Emby client factory + implementations
│   ├── geoip.ts      # MaxMind GeoIP lookups
│   ├── cache.ts      # Redis cache layer
│   ├── sync.ts       # Server sync orchestration
│   └── ...
├── jobs/             # Background jobs via BullMQ (Redis-backed queues)
├── plugins/          # Fastify plugins (auth, rate limit, etc.)
├── utils/            # Shared utility functions
├── constants/        # Server-side constants
└── websocket/        # Socket.io real-time event handling
```

**Key dependency chain:**

```
HTTP Request → Route Handler → Service Layer → DB (Drizzle) / Cache (Redis) / Media Server Client
                                              ↑
                                    BullMQ Jobs (async, queued)
```

---

### The Web App (`apps/web`)

The frontend is a **React 19** SPA using **Vite** as the bundler.

```
src/
├── App.tsx           # Root router — React Router 7 routes
├── main.tsx          # Entry point — wraps App with providers
├── pages/            # One component per route (Dashboard, Users, Rules, etc.)
├── components/
│   ├── ui/           # shadcn/ui primitives (Button, Dialog, Table, etc.)
│   ├── layout/       # App shell (sidebar, header)
│   ├── sessions/     # Session-specific components
│   ├── rules/        # Rule builder components
│   └── ...           # Feature-specific component folders
├── hooks/
│   ├── useAuth.tsx   # Auth context + React Query integration
│   ├── useSocket.tsx # Socket.io context for real-time events
│   ├── queries/      # React Query hooks (one file per domain)
│   └── ...
└── lib/
    ├── api.ts        # Typed API client (wraps fetch)
    ├── basePath.ts   # Base path utility
    └── formatters.ts # Date, duration, file size formatters
```

---

### The Shared Package (`packages/shared`)

This is the **contract layer** — everything shared between server and web lives here:

- **`types.ts`** — TypeScript interfaces (`Session`, `Rule`, `User`, `Server`, etc.)
- **`schemas.ts`** — Zod validation schemas (used by the server for request validation and by
  the web for form validation)
- **`constants.ts`** — Rule defaults, severity levels, media types
- **`violations.ts`** — Violation calculation logic

If you add a new API endpoint, you'll often need to:

1. Add/update types in `packages/shared/src/types.ts`
2. Add a Zod schema in `packages/shared/src/schemas.ts`
3. Implement the route in `apps/server/src/routes/`
4. Add the API call in `apps/web/src/lib/api.ts`
5. Add a React Query hook in `apps/web/src/hooks/queries/`

---

## Part 4 — Tech Stack Deep Dive

### Backend Stack

| Technology        | Role              | Why                                                                                     |
| ----------------- | ----------------- | --------------------------------------------------------------------------------------- |
| **Fastify 5**     | HTTP framework    | Faster than Express, plugin architecture, built-in TypeScript support                  |
| **Drizzle ORM**   | Database ORM      | Type-safe SQL, explicit queries, great TypeScript inference                             |
| **TimescaleDB**   | Database          | PostgreSQL + time-series extensions — critical for session history analytics at scale   |
| **Redis / BullMQ**| Cache + job queue | Session state caching, background job processing (library syncs, GeoIP updates)         |
| **Socket.io**     | Real-time events  | Session start/stop/update events pushed to dashboard in real-time                       |
| **Zod**           | Validation        | Runtime type-checking for request bodies — schemas live in `@tracearr/shared`          |

### Frontend Stack

| Technology            | Role            | Why                                                                              |
| --------------------- | --------------- | -------------------------------------------------------------------------------- |
| **React 19**          | UI framework    | Latest concurrent features, improved Suspense                                    |
| **Vite 7**            | Build tool      | Near-instant HMR, fast builds                                                    |
| **React Router 7**    | Routing         | File-system-friendly routing                                                     |
| **TanStack Query v5** | Server state    | Automatic caching, background refetch, cache invalidation via Socket.io events   |
| **Tailwind CSS v4**   | Styling         | Utility-first, JIT compilation                                                   |
| **shadcn/ui**         | Components      | Radix UI primitives + Tailwind — copy-paste components you own                   |
| **Highcharts**        | Charts          | Analytics dashboards (activity, bandwidth, devices)                              |
| **Leaflet**           | Maps            | GeoIP session maps                                                               |

### Mobile Stack

React Native via **Expo** (SDK 55), using Expo Router for file-based navigation. Shares types
and schemas from `@tracearr/shared`. Uses its own API client built on Axios.

---

## Part 5 — Data Model

Understanding the data model is fundamental. Read the schema comment at the top of
`apps/server/src/db/schema.ts`:

```
Multi-Server User Architecture:
- `users`        = Identity (the real human)
- `server_users` = Account on a specific server (Plex/Jellyfin/Emby)
- One user can have multiple server_users (accounts across servers)
- Sessions and violations link to server_users (server-specific)
```

Key tables and their relationships:

```
servers          → Physical Plex/Jellyfin/Emby instances
  └── server_users  → User accounts on that server
        ├── sessions    → Streaming sessions (with GeoIP, device, media info)
        └── violations  → Rule violations attributed to this server user

users            → Identity layer (the real person behind server accounts)
  └── server_users  (one-to-many)

rules            → Access control rules (impossible_travel, concurrent_streams, etc.)
  └── violations  → When a rule fires on a session
```

Sessions are stored in **TimescaleDB hypertables** — this enables efficient time-range queries
for analytics (e.g., "all sessions in the last 30 days grouped by hour") without full table scans.

---

## Part 6 — Design Philosophy & Patterns

### 1. Shared-First Typing

**Never define a type in `apps/server` or `apps/web` that could belong in `packages/shared`.**
Types that cross the API boundary — request/response shapes, domain entities — live in shared.
This ensures the frontend and backend are always in sync.

```typescript
// ✅ DO: import from @tracearr/shared in both server and web
import type { Session, Rule, ActiveSession } from '@tracearr/shared';
import { sessionQuerySchema } from '@tracearr/shared';

// ❌ DON'T: define duplicated types locally
interface Session { ... } // local copy will drift
```

### 2. Zod for the Full Validation Cycle

Zod schemas in `@tracearr/shared` serve double duty — server-side request validation AND
frontend form validation via `@hookform/resolvers/zod`. This eliminates validation drift.

```typescript
// packages/shared/src/schemas.ts
export const createServerSchema = z.object({
  name: z.string().min(1).max(100),
  type: z.enum(['plex', 'jellyfin', 'emby']),
  url: z.string().url(),
  token: z.string().min(1),
});

// apps/server/src/routes/servers.ts — validates incoming request
app.post('/', {
  schema: { body: zodToJsonSchema(createServerSchema) }
}, async (request) => { ... });

// apps/web — validates form input before submit
const form = useForm({ resolver: zodResolver(createServerSchema) });
```

### 3. Route → Service Separation

Routes are thin. They handle HTTP concerns (parsing, auth, response shaping) and delegate logic
to services. Services are testable in isolation.

```typescript
// routes/servers.ts — thin route
app.post('/', { preHandler: [app.authenticate] }, async (request) => {
  const body = createServerSchema.parse(request.body);
  const server = await syncServer(body); // delegate to service
  return server;
});

// services/sync.ts — business logic
export async function syncServer(config: CreateServerInput): Promise<Server> {
  const client = createMediaServerClient(config); // factory pattern
  const serverInfo = await client.getServerInfo();
  // ... database writes, cache invalidation, etc.
}
```

### 4. The Media Server Factory Pattern

Plex, Jellyfin, and Emby all look different under the hood. The codebase uses a
**factory + interface** pattern to make callers agnostic:

```typescript
// Usage — caller doesn't care if it's Plex or Jellyfin
const client = createMediaServerClient({
  type: server.type, // 'plex' | 'jellyfin' | 'emby'
  url: server.url,
  token: server.token,
});

const sessions = await client.getSessions(); // same API regardless
const users = await client.getUsers();
```

When adding support for a new media server feature, implement `IMediaServerClient` and add the
case to the factory. Everything else just works.

### 5. React Query for ALL Server State

The web app uses **TanStack Query** for everything that comes from the API. No Redux, no Context
for data fetching — just query hooks. Real-time updates from Socket.io invalidate query cache
entries:

```typescript
// hooks/queries/sessions.ts — define the query
export function useActiveSessions(serverId?: string) {
  return useQuery({
    queryKey: ['sessions', 'active', serverId],
    queryFn: () => api.sessions.getActive({ serverId }),
    staleTime: 30_000,
  });
}

// hooks/useSocket.tsx — Socket.io event invalidates cache
socket.on('session:started', (session) => {
  queryClient.invalidateQueries({ queryKey: ['sessions', 'active'] });
  // UI refreshes automatically — no manual state management
});
```

**Query key convention:** `[domain, subtype, ...filters]`
Examples: `['sessions', 'active', serverId]`, `['stats', 'dashboard']`, `['users', userId]`.

### 6. Role-Based Access (Owner vs. Guest)

Every authenticated request carries `request.user` with a `role` property (`'owner'` or
`'guest'`) and a `serverIds` array (which servers the user can see). Owners see everything;
guests are scoped to their servers. Always check this in routes:

```typescript
.where(
  authUser.role === 'owner'
    ? undefined                              // Owners see all
    : inArray(servers.id, authUser.serverIds) // Guests are scoped
)
```

### 7. Background Jobs via BullMQ

Long-running work — library syncs, GeoIP database updates, Tautulli imports — goes through
BullMQ job queues backed by Redis. Never do heavy work inline in a route handler.

```typescript
// Enqueue a library sync job (non-blocking)
await enqueueLibrarySync({ serverId: server.id });

// The actual work happens in a BullMQ worker
// Progress is emitted via Socket.io to the frontend
```

### 8. Real-Time Updates via Socket.io

Events flow:
**Backend job/service → Socket.io emit → Frontend socket handler → React Query invalidation → UI re-render**

The full event type list lives in `@tracearr/shared` as `WS_EVENTS` constants. Always use the
constants, never hardcode event strings.

---

## Part 7 — Testing Strategy

The project has a layered test architecture:

| Layer           | Command                | What it tests                               |
| --------------- | ---------------------- | ------------------------------------------- |
| **Unit**        | `pnpm test:unit`       | Pure functions, utilities, schemas          |
| **Services**    | `pnpm test:services`   | Service logic with mocked DB                |
| **Routes**      | `pnpm test:routes`     | HTTP handlers with mocked services          |
| **Security**    | `pnpm test:security`   | Auth bypass, injection, rate limit          |
| **Integration** | `pnpm test:integration`| Real DB + Redis (requires `docker:up`)      |
| **E2E**         | `pnpm test:e2e`        | Full stack browser tests                    |

**Run `test:unit` constantly.** They're fast (< 5s) and catch regressions immediately. Run the
full suite before opening a PR.

Test factories and shared mocks live in `packages/test-utils/` — use them instead of writing
boilerplate setup per test.

---

## Part 8 — Your First Contribution

### Adding a new dashboard stat

1. Add the type to `packages/shared/src/types.ts`
2. Add the DB query in `apps/server/src/services/dashboardStats.ts`
3. Expose it via a route in `apps/server/src/routes/stats/`
4. Add the API call to `apps/web/src/lib/api.ts`
5. Add a `useQuery` hook in `apps/web/src/hooks/queries/`
6. Drop a Highcharts chart component into the appropriate page

### Adding a new rule type

1. Add to the `ruleTypeEnum` in `apps/server/src/db/schema.ts`, then run `db:generate`
2. Add default params to `RULE_DEFAULTS` in `packages/shared/src/constants.ts`
3. Implement the evaluation logic as a method on `RuleEngine` in
   `apps/server/src/services/rules.ts`
4. Add the UI form fields in `apps/web/src/components/rules/`

---

## Quick Reference

```bash
# Spin up from scratch
pnpm docker:up
pnpm install
pnpm --filter @tracearr/server db:migrate
pnpm dev

# Day-to-day
pnpm test:unit      # Fast tests — run constantly
pnpm typecheck      # Full type check
pnpm lint:fix       # Auto-fix lint issues

# Database
pnpm --filter @tracearr/server db:generate   # After schema.ts changes
pnpm --filter @tracearr/server db:migrate    # Apply migrations
pnpm --filter @tracearr/server db:studio     # Browse DB visually

# Get help
# GitHub Discussions → for design questions
# Discord            → for quick questions (link in CONTRIBUTING.md)
```

---

Welcome to the team — dive in, ask questions on Discord, and ship something cool. 🎬
