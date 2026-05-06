# Workspace

## Overview

Full-stack author-owned publishing platform deployed on Replit as a single Express process that serves both the frontend and the API.

## Stack

- **Monorepo tool**: npm workspaces (npm 11.12.1 — pnpm is not used anywhere)
- **Node.js version**: 24
- **Package manager**: npm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: MySQL via Drizzle ORM + `mysql2` (driver). Connection configured via `DB_HOST`/`DB_PORT`/`DB_NAME`/`DB_USER`/`DB_PASS`/`DB_SSL`.
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec → React Query hooks + Zod schemas)
- **Build**: esbuild (API server, ESM bundle) + Vite (frontend)
- **Auth**: Auth.js with GitHub + Google OAuth, local sessions, and app-owned roles
- **Frontend**: React + Vite (Tailwind CSS)

## Packages

| Package | Path | Purpose |
|---|---|---|
| `@workspace/api-server` | `artifacts/api-server/` | Express API server (posts, comments, users, feed stats) |
| `@workspace/microblog` | `artifacts/microblog/` | React + Vite frontend (home feed, post detail, user profile) |
| `@workspace/db` | `lib/db/` | Drizzle schema + db client (MySQL via `mysql2`) |
| `@workspace/api-spec` | `lib/api-spec/` | OpenAPI 3.1 spec + Orval codegen config |
| `@workspace/api-client-react` | `lib/api-client-react/` | Generated React Query hooks + custom fetch |
| `@workspace/api-zod` | `lib/api-zod/` | Generated Zod request/response schemas |

## Key Commands

- `npm run typecheck` — full typecheck across all packages
- `npm run build` — typecheck + build all packages
- `npm run codegen --workspace=@workspace/api-spec` — regenerate API hooks and Zod schemas from OpenAPI spec
- `npm run push --workspace=@workspace/db` — push DB schema changes manually when needed
- `npm run push-force --workspace=@workspace/db` — force Drizzle schema push
- `npm run dev` — default single-port local app flow
- `npm run dev:hot` — optional two-port dev flow with Vite hot reload
- `npm run dev:api` — run API server locally
- `npm run dev:web` — run the Vite frontend locally on `FRONTEND_PORT` (or `PORT` when launched as a Replit artifact)
- `npm run list-users --workspace=@workspace/scripts` — list local users after first sign-in
- `npm run promote-owner --workspace=@workspace/scripts -- --email you@example.com` — promote your account to owner

## Database

MySQL is the canonical datastore for both local development and the deployed app. Current tables include:

- `users`, `accounts`, `sessions`, `verification_tokens`
- `user_ai_vendor_settings`
- `posts`, `comments`, `reactions`
- `feed_sources`, `feed_items_seen`
- `categories`, `post_categories`
- `pages`, `nav_links`, `site_settings`

Drizzle schema lives in `lib/db/src/schema/`. The runtime reconciliation path is `lib/db/src/migrate.ts`, which is the current source of truth for the shipped schema. `npm run push --workspace=@workspace/db` is available, but the deployed app also relies on the runtime schema shape matching `ensureTables()`.

## API Routes

- `GET /api/healthz` — health check
- `GET /api/posts` — list published posts (paginated); optional `?category=<slug|uncategorized>` and `?source=<id|original>` server-side filters
- `POST /api/posts` — create post (owner only)
- `GET /api/posts/:id` — get post + comments
- `PATCH /api/posts/:id` — update post (owner only)
- `DELETE /api/posts/:id` — delete post (owner only)
- `GET /api/posts/user/:userId` — get user's posts
- `GET /api/posts/search` — public post search
- `GET /api/posts/pending` — owner moderation queue for pending imported posts
- `POST /api/posts/:id/approve` — owner approval for pending imported post
- `POST /api/posts/:id/reject` — owner rejection for pending imported post
- `POST /api/posts/:postId/comments` — add comment (signed-in users)
- `PATCH /api/comments/:id` — update comment (author or owner)
- `DELETE /api/comments/:id` — delete comment (author or owner)
- `GET /api/users/me` — current user profile (auth required)
- `PATCH /api/users/me` — update current user profile/theme
- `GET/PATCH /api/users/me/ai-settings` — owner AI vendor settings
- `POST /api/ai/process` — owner AI text processing
- `GET/PATCH /api/site-settings` — site settings
- `GET/POST/PATCH/DELETE /api/categories...` — category management
- `GET/POST/PATCH/DELETE /api/pages...` — CMS pages
- `GET/POST/PATCH/DELETE /api/feed-sources...` — inbound feed source management
- `GET /api/feed-sources/public` — public list of active inbound feed sources
- `GET /api/feeds` — public feed catalog
- `GET /api/feed/stats` — total posts + comments count

## Auth.js

- Backend auth is mounted at `/api/auth/*` in the Express server
- Default local development uses one origin at `http://localhost:8080`
- Optional hot mode uses frontend `http://localhost:3000` with API/Auth at `http://localhost:8080`
- The frontend dev server proxies both `/api/*` and `/api/auth/*` to the backend
- The web app uses cookie-backed sessions; do not attach bearer tokens for browser API calls
- The first owner is promoted manually after first login using the scripts package
- In production on Replit, the built frontend and the API share one origin

## Important Notes

- `mysql2` is bundled via esbuild for the API server; native modules are listed as externals in `artifacts/api-server/build.mjs`.
- Route order in `posts.ts`: `/feed/stats` and `/posts/user/:userId` come BEFORE `/posts/:id`.
- Route order in `routes/index.ts`: pending-post routes mount before generic post routes; pages mount after categories to avoid route collisions.
- Drizzle operators (`eq`, `desc`, `count`, etc.) are re-exported from `@workspace/db` to avoid version conflicts.
- The API server handles `SIGTERM`/`SIGINT` gracefully (idempotent shutdown with a 5s force-exit safeguard) so workflow restarts and deploys exit cleanly.
- `AI_SETTINGS_ENCRYPTION_KEY` must decode to exactly 32 bytes. A plain 32-character ASCII string is valid.
- Optional site identity env vars: `PUBLIC_SITE_URL` (canonical origin used in feed links and OG tags — set this in production), `SITE_TITLE`, `SITE_DESCRIPTION`, `SITE_AUTHOR_NAME`.
- Feed sources have an optional `author_name` column that overrides the byline on all imported posts from that source. Manageable via `/admin/feeds` inline edit. Imported post cards show the source blog name as the primary byline; when the feed item has a distinct individual author, the attribution line shows "by [author] via [Blog]".
- Display name changes (`PATCH /api/users/me`) retroactively update `author_name` on all existing posts by that user.
- `GET /api/feeds` (feed catalog) always returns all category feeds (Atom + JSON) in the default response; the former `?category=` filter param is kept as a no-op for backwards compatibility. The frontend `/feeds` page groups feeds into sections.
- `artifacts/microblog/vite.config.ts`:
  - Listens on `FRONTEND_PORT ?? PORT ?? 3000` so it works both locally and inside the Replit artifact (which sets `PORT`).
  - Proxies `/api/*` and `/api/auth/*` to `API_ORIGIN` (default `http://localhost:${API_PORT ?? 8080}`). Use `API_PORT`, **not** `PORT`, when overriding — `PORT` is the frontend's own port.

## Deployment

- Configured in `.replit` under `[deployment]`:
  - `deploymentTarget = "autoscale"`
  - `build = ["npm", "run", "build"]` — runs typecheck + Vite + esbuild across all workspaces.
  - `run = ["node", "--enable-source-maps", "artifacts/api-server/dist/index.mjs"]` — single-process server that serves the built frontend statically from `artifacts/microblog/dist/public` and the API on `/api/*` on the same port.
- Deployment uses **npm** end-to-end. There are no pnpm invocations in any `artifact.toml` or root config.

Use the root `package.json` workspace configuration for workspace structure, TypeScript setup, and package details.
