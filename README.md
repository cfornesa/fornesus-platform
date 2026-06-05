# fornesus-platform

Author-owned social publishing platform deployed on Replit.

The current shipped app is a TypeScript npm-workspaces monorepo with:

- an Express 5 API server
- a React 19 + Vite frontend
- Auth.js sign-in with local app-owned roles
- MySQL via Drizzle ORM

## What The Site Does

- The site owner publishes canonical posts.
- Signed-in members can comment on published posts.
- Comment authors and the owner can edit or delete comments.
- Display name changes propagate retroactively to the `author_name` on all existing posts by that user.
- The owner can manage categories, standalone pages, site settings, nav links, inbound feed subscriptions, pending imported posts, draft/scheduled posts, platform syndication connections, media assets, interactive art pieces, exhibits, and AI vendor settings.
- Inbound feed sources support an optional per-source author name override that controls the displayed author on all posts imported from that source.
- Public feeds are available at `GET /api/feeds/atom`, `GET /api/feeds/json`, `GET /api/feeds/mf2`, `GET /feed.xml`, `GET /feed.json`, `GET /export/json`, and `GET /export.json`.
- Category feeds and page feeds are published through proxy-safe `/api` routes with legacy aliases kept functional.
- Owner AI writing assistance is available through saved vendor settings in `user_ai_vendor_settings`.
- Owner AI can also generate and validate reusable `p5`, `c2`, and `three` interactive art pieces.
- Rich posts can include uploaded/imported media, featured images, embedded pieces, embedded exhibits, and immersive-view links.

## Runtime Shape

Replit deployment is the canonical production shape.

- Build: `npm run build`
- Run: `node --enable-source-maps artifacts/api-server/dist/index.mjs`
- The API server serves both the built frontend and `/api/*` from one process.
- Auth.js is mounted at `/api/auth/*`.
- Static frontend output is served from `artifacts/microblog/dist/public`.

Local development has two supported modes:

1. `npm run dev`
   This matches deployment most closely. The API server serves the built frontend and all auth/API routes on one origin, using `PORT` from the environment and falling back to `8080` when unset.
2. `npm run dev:hot`
   Vite serves the frontend on `FRONTEND_PORT` and proxies `/api/*` and `/api/auth/*` to the API server on `API_ORIGIN` or the configured API port.

## Stack

- Monorepo: npm workspaces
- Language: TypeScript
- API: Express 5
- Frontend: React 19, Vite, Tailwind CSS
- Database: MySQL with `mysql2` + Drizzle ORM
- Auth: Auth.js with GitHub and Google OAuth
- Validation/codegen: Zod, drizzle-zod, Orval
- Rich editor: TipTap
- HTML sanitization: `sanitize-html`
- Feed ingest: `rss-parser`
- Creative runtimes: `p5`, `c2.js`, `three`
- Dynamic OG images: Satori + Resvg

## Key Commands

```bash
npm run dev
npm run dev:hot
npm run dev:api
npm run dev:web
npm run typecheck
npm run build
npm run list-users --workspace=@workspace/scripts
npm run promote-owner --workspace=@workspace/scripts -- --email you@example.com
```

## Environment

Important runtime variables include:

```env
PORT=8080
FRONTEND_PORT=3000
ALLOWED_ORIGINS=http://localhost:8080
AUTH_SECRET=replace_with_a_long_random_secret
SESSION_SECRET=replace_with_a_long_random_secret
GITHUB_ID=your_github_oauth_app_client_id
GITHUB_SECRET=your_github_oauth_app_client_secret
GOOGLE_CLIENT_ID=your_google_oauth_client_id
GOOGLE_CLIENT_SECRET=your_google_oauth_client_secret
DB_HOST=your_mysql_host
DB_PORT=3306
DB_NAME=your_database_name
DB_USER=your_database_user
DB_PASS=your_database_password
DB_SSL=false
CRON_SECRET=replace_with_a_long_random_secret
AI_SETTINGS_ENCRYPTION_KEY=12345678901234567890123456789012

# Optional — used by two-port hot development
# API_ORIGIN=http://localhost:8080

# Optional — set in production so feed links and OG tags always use the right origin
# PUBLIC_SITE_URL=https://your-domain.com
# SITE_TITLE=My Microblog
# SITE_DESCRIPTION=A personal microblog.
# SITE_AUTHOR_NAME=Your Name
```

`AI_SETTINGS_ENCRYPTION_KEY` must decode to exactly `32 bytes`. A plain 32-character ASCII string is valid. The API server will throw if this value is missing or the decoded size is not exactly 32 bytes.

## Database

MySQL is the canonical datastore for both deployment and local authoring.

Current live schema includes:

- `users`, `accounts`, `sessions`, `verification_tokens`
- `user_ai_vendor_settings` — named-profile model as of 2026-06-01; each row now has an auto-increment `id` PK, a `profile_name` column, and an `endpoint_kind` column. Users reference profiles by ID via `preferred_art_piece_profile_id`, `preferred_text_improve_profile_id`, `preferred_alt_text_profile_id`. The old vendor-string preference columns on `users` have been dropped.
- `posts`, `comments`, `reactions`
- `feed_sources`, `feed_items_seen`
- `categories`, `post_categories`
- `pages`, `nav_links`, `site_settings`
- `platform_connections`, `post_syndications`, `platform_oauth_apps`
- `media_assets`, `profile_photo_assets`, `site_assets`
- `art_pieces`, `art_piece_versions`
- `exhibits`, `piece_exhibits`, `media_asset_exhibits`
- `site_bootstrap_state`

The repo has two schema references:

- [lib/db/src/migrate.ts](/Users/Fornesus/Code/fornesus-platform/lib/db/src/migrate.ts:100)
  This is the runtime reconciliation path used by the API server.
- [lib/db/install.sql](/Users/Fornesus/Code/fornesus-platform/lib/db/install.sql:1)
  This is the fresh-install SQL for environments where you cannot run the Node app first.

For the current shipped app, treat the runtime schema in `lib/db/src/migrate.ts` as the source of truth. Do not use older cleanup guidance that removes `categories`, `pages`, `site_settings`, feed tables, per-user theme columns, platform syndication tables, media assets, art pieces, or exhibits from a live current deployment.

## Important Routes

Public and auth:

- `GET /api/healthz`
- `GET /api/bootstrap-status`
- `GET /api/posts` — accepts optional `?category=<slug|uncategorized>` and `?source=<id|original>` server-side filters
- `GET /api/posts/drafts` owner only
- `GET /api/posts/:id`
- `GET /api/posts/search`
- `GET /api/posts/user/:userId`
- `GET /api/users/:id`
- `GET /api/nav-links`
- `GET /api/site-settings`
- `GET /api/categories`
- `GET /api/categories/:slug`
- `GET /api/categories/:slug/posts`
- `GET /api/pages`
- `GET /api/pages/:slug`
- `GET /api/feed-sources/public`
- `GET /api/feeds`
- `GET /api/feeds/atom`
- `GET /api/feeds/json`
- `GET /api/feeds/mf2`
- `GET /api/categories/:slug/feeds/atom`
- `GET /api/categories/:slug/feeds/json`
- `GET /api/p/:slug/feeds/atom`
- `GET /api/p/:slug/feeds/json`
- `GET /feed.xml`
- `GET /feed.json`
- `GET /export/json`
- `GET /export.json`
- `GET /atom`
- `GET /jsonfeed`
- `GET /embed/pieces/:id`
- `GET /immersive/images/:encodedRef`
- `GET /immersive/pieces/:id`
- `GET /immersive/exhibits/:slug`

Owner-managed or authenticated routes:

- `POST /api/posts` owner only
- `PATCH /api/posts/:id` owner only
- `DELETE /api/posts/:id` owner only
- `POST /api/posts/:postId/comments` signed-in users
- `PATCH /api/comments/:id` comment author or owner
- `DELETE /api/comments/:id` comment author or owner
- `GET /api/users/me`
- `PATCH /api/users/me`
- `GET /api/users/me/ai-settings` owner only
- `PATCH /api/users/me/ai-settings` owner only
- `POST /api/ai/process` owner only
- `POST /api/ai/describe-image` owner only
- `GET|POST|PATCH|DELETE /api/media...` owner only, except public `GET /api/media/:fileName`
- `GET|POST|PATCH|DELETE /api/art-pieces...` owner only, except public embed reads
- `GET|POST|PATCH|DELETE /api/exhibits...` owner-managed writes with public reads
- `PUT /api/art-pieces/:id/exhibits` owner only
- `PUT /api/media/:fileName/exhibits` owner only
- `PATCH /api/site-settings` owner only
- `POST|PATCH|DELETE /api/categories...` owner only
- `POST|PATCH|DELETE /api/pages...` owner only
- `POST|PATCH|DELETE /api/nav-links...` owner only
- `GET|POST|PATCH|DELETE /api/feed-sources...` owner only, except `/feed-sources/public`
- `GET /api/posts/pending` owner only
- `POST /api/posts/:id/approve` owner only
- `POST /api/posts/:id/reject` owner only
- `GET|PUT /api/platform-oauth-apps...` owner only
- `GET|POST|PATCH|DELETE /api/platform-connections...` owner only
- `GET|POST /api/platform-oauth...` owner only
- `POST /api/bootstrap/complete` owner only
- `GET /api/recycle-bin` owner only
- `POST /api/recycle-bin/posts/:id/restore` owner only
- `DELETE /api/recycle-bin/posts/:id` owner only (permanent delete)
- `POST /api/recycle-bin/pieces/:id/restore` owner only
- `DELETE /api/recycle-bin/pieces/:id` owner only (permanent delete)
- `POST /api/recycle-bin/media/:id/restore` owner only
- `DELETE /api/recycle-bin/media/:id` owner only (permanent delete)
- `POST /api/recycle-bin/exhibits/:id/restore` owner only
- `DELETE /api/recycle-bin/exhibits/:id` owner only (permanent delete)
- `POST /api/recycle-bin/pages/:id/restore` owner only
- `DELETE /api/recycle-bin/pages/:id` owner only (permanent delete)
- `POST /api/recycle-bin/categories/:id/restore` owner only
- `DELETE /api/recycle-bin/categories/:id` owner only (permanent delete)
- `DELETE /api/recycle-bin` owner only (empties entire bin permanently)

## Owner Bootstrap

1. Start the app.
2. Sign in once with the account that should own the site.
3. Promote that account:

```bash
npm run promote-owner --workspace=@workspace/scripts -- --email you@example.com
```

## Replit Notes

- Replit preview/dev URLs and deployed `.replit.app` origins are both supported by the API server CORS logic when the request host matches the origin.
- In deployment, the app is intended to run as a single server process.
- For OAuth on Replit, configure callbacks against the actual exposed origin for the environment you are using.
- Canonical origin generation uses the first `ALLOWED_ORIGINS` entry, then `PUBLIC_SITE_URL`, then request headers, then `https://meet.fornesus.com`.

## Docs Map

- [replit.md](replit.md) for operational repo/runtime notes
- [docs/auth-setup.md](docs/auth-setup.md) for local, Replit, and OAuth setup
- [docs/dependencies.md](docs/dependencies.md) for vendor/dependency tradeoffs
- [docs/ai-vendor-verification.md](docs/ai-vendor-verification.md) for AI vendor runbook
- [docs/db-cleanup-report.md](docs/db-cleanup-report.md) — historical only; the old cleanup guidance is superseded
