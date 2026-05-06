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
- The owner can manage categories, standalone pages, site settings, nav links, inbound feed subscriptions, pending imported posts, and AI vendor settings.
- Public feeds are available at `GET /feed.xml`, `GET /feed.json`, `GET /export/json`, and `GET /export.json`.
- Category feeds and page feeds are also published.
- Owner AI writing assistance is available through saved vendor settings in `user_ai_vendor_settings`.

## Runtime Shape

Replit deployment is the canonical production shape.

- Build: `npm run build`
- Run: `node --enable-source-maps artifacts/api-server/dist/index.mjs`
- The API server serves both the built frontend and `/api/*` from one process.
- Auth.js is mounted at `/api/auth/*`.
- Static frontend output is served from `artifacts/microblog/dist/public`.

Local development has two supported modes:

1. `npm run dev`
   This matches deployment most closely. The API server serves the built frontend and all auth/API routes on one origin, usually `http://localhost:8080`.
2. `npm run dev:hot`
   Vite serves the frontend on `http://localhost:3000` and proxies `/api/*` and `/api/auth/*` to the API server on `http://localhost:8080`.

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
API_ORIGIN=http://localhost:8080
ALLOWED_ORIGINS=http://localhost:8080
AUTH_SECRET=replace_with_a_long_random_secret
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
```

`AI_SETTINGS_ENCRYPTION_KEY` must decode to exactly `32 bytes`. A plain 32-character ASCII string is valid. The API server will throw if this value is missing or the decoded size is not exactly 32 bytes.

## Database

MySQL is the canonical datastore for both deployment and local authoring.

Current live schema includes:

- `users`, `accounts`, `sessions`, `verification_tokens`
- `user_ai_vendor_settings`
- `posts`, `comments`, `reactions`
- `feed_sources`, `feed_items_seen`
- `categories`, `post_categories`
- `pages`, `nav_links`, `site_settings`

The repo has two schema references:

- [lib/db/src/migrate.ts](/Users/Fornesus/Code/fornesus-platform/lib/db/src/migrate.ts:100)
  This is the runtime reconciliation path used by the API server.
- [lib/db/install.sql](/Users/Fornesus/Code/fornesus-platform/lib/db/install.sql:1)
  This is the fresh-install SQL for environments where you cannot run the Node app first.

For the current shipped app, treat the runtime schema in `lib/db/src/migrate.ts` as the source of truth. Do not use older cleanup guidance that removes `categories`, `pages`, `site_settings`, feed tables, or per-user theme columns from a live current deployment.

## Important Routes

Public and auth:

- `GET /api/healthz`
- `GET /api/posts`
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
- `GET /feed.xml`
- `GET /feed.json`
- `GET /export/json`
- `GET /export.json`

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
- `PATCH /api/site-settings` owner only
- `POST|PATCH|DELETE /api/categories...` owner only
- `POST|PATCH|DELETE /api/pages...` owner only
- `POST|PATCH|DELETE /api/nav-links...` owner only
- `GET|POST|PATCH|DELETE /api/feed-sources...` owner only, except `/feed-sources/public`
- `GET /api/posts/pending` owner only
- `POST /api/posts/:id/approve` owner only
- `POST /api/posts/:id/reject` owner only

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

## Docs Map

- [replit.md](/Users/Fornesus/Code/fornesus-platform/replit.md:1) for operational repo/runtime notes
- [docs/auth-setup.md](/Users/Fornesus/Code/fornesus-platform/docs/auth-setup.md:1) for local, Replit, and OAuth setup
- [docs/dependencies.md](/Users/Fornesus/Code/fornesus-platform/docs/dependencies.md:1) for vendor/dependency tradeoffs
- [docs/ai-vendor-verification.md](/Users/Fornesus/Code/fornesus-platform/docs/ai-vendor-verification.md:1) for AI vendor runbook
