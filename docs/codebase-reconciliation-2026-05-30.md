# Codebase Reconciliation — 2026-05-30

This audit backfills markdown from repository evidence after a series of feature commits landed without matching memory or operations documentation.

## Scope

Evidence reviewed:

- Git history from `2026-05-07` through `2026-05-29`.
- Current OpenAPI paths in `lib/api-spec/openapi.yaml`.
- Current runtime schema reconciliation in `lib/db/src/migrate.ts`.
- Current route implementations under `artifacts/api-server/src/routes`.
- Current frontend route/admin/editor implementations under `artifacts/microblog/src`.
- Current uncommitted immersive-view changes in the working tree.

This document records what is implemented. It does not authorize new product direction, URL migrations, auth endpoint changes, or new vendors.

## Shipped Changes Not Fully Reflected In Markdown

### Posts, Drafts, Scheduling, And Syndication

- Posts now have four statuses: `published`, `pending`, `draft`, and `scheduled`.
- Owner-created drafts are listed separately through `GET /api/posts/drafts`.
- Scheduled posts store `scheduled_at`; the in-process `post-scheduler.ts` checks due posts every 60 seconds, publishes them, and dispatches any deferred syndication targets.
- Draft or scheduled posts can store `pending_platform_ids`, which are cleared after syndication is dispatched on publish.
- Posts can store `featured_image_url` and per-platform `social_post_drafts`.
- The Admin Posts page now exposes draft management and a publishing calendar.

### POSSE Platform Surface

- Platform connection storage now covers WordPress.com, self-hosted WordPress, Medium, Blogger, Substack, Bluesky, LinkedIn, Facebook, and Instagram.
- `platform_oauth_apps` stores encrypted app credentials for WordPress.com, Blogger, LinkedIn, and Meta/Facebook.
- Bluesky uses an AT Protocol App Password rather than a developer OAuth app.
- LinkedIn uses the Posts API with `w_member_social`.
- Facebook and Instagram share the Meta OAuth app flow; Instagram requires a linked Business/Creator account and a featured image for publishing.
- Article-style syndication targets include a visible source footer; social targets use canonical URLs in platform-native text/cards/captions.

### Local Media Library

- `media_assets` now stores uploaded/imported image bytes in MySQL with filename, URL, title, MIME type, alt text, and upload time.
- Owner-only media routes support upload, URL import, listing, metadata update, deletion, and public file reads.
- Rich posts can insert media from the library, edit image metadata, and use the first uploaded content image as the featured image unless the owner selected one manually.
- Upload/import size remains capped at 8 MB.

### AI Vendors And AI Tasks

- Supported owner-configured AI vendors now include OpenRouter, OpenCode Zen, OpenCode Go, Google Gemini, Mistral AI, Mistral Vibe, and DeepSeek.
- Text generation is available for all configured vendors.
- Image alt text generation excludes DeepSeek until image-input support is verified.
- Interactive piece generation is limited to Google, Mistral AI, Mistral Vibe, and DeepSeek.
- DeepSeek piece generation uses a larger output budget and non-thinking mode diagnostics.

### Interactive Art Pieces

- `art_pieces` and `art_piece_versions` store reusable owner-created interactive pieces.
- Supported engines are `p5`, `c2`, and `three`.
- Pieces can be created manually or generated through configured AI vendors.
- AI-generated drafts must pass bounded validation/preflight before they can be saved.
- Saved versions include HTML, CSS, JavaScript, engine, generation vendor/model, validation status, attempt count, and notes.
- Piece embeds are exposed through `/embed/pieces/:id` and `/api/art-pieces/:id/embed`.
- Runtime libraries are served under `/api/runtimes/*` so Replit proxy behavior does not intercept JavaScript runtime assets.

### Immersive Views And Exhibits

- Featured images can open in `/immersive/images/:encodedRef`.
- Art pieces can open in `/immersive/pieces/:id`.
- Exhibits can open in `/immersive/exhibits/:slug`.
- Exhibits are owner-managed collections of art pieces and media assets with configurable `rows`, `cols`, `artist_statement`, and `biography`.
- Exhibit membership tables are `piece_exhibits` and `media_asset_exhibits`.
- Rich posts can insert saved exhibit embeds.
- Exhibit walls can render interactive and static iframe variants.
- Current uncommitted changes improve immersive fullscreen behavior, visual viewport sizing, scroll/touch locking, and resize behavior without resetting camera position.

### Feeds And Origin Resolution

- Replit proxy-safe feed content routes are under `/api`: `/api/feeds/atom`, `/api/feeds/json`, `/api/feeds/mf2`, `/api/categories/:slug/feeds/*`, and `/api/p/:slug/feeds/*`.
- Legacy public aliases remain implemented: `/feed.xml`, `/feed.json`, `/export/json`, `/export.json`, `/atom`, `/jsonfeed`, and per-category/page extension aliases.
- Feed and embed origin resolution now flows through `getCanonicalOrigin()`: first `ALLOWED_ORIGINS`, then `PUBLIC_SITE_URL`, then request protocol/host, then `https://meet.fornesus.com`.
- This supersedes earlier docs that said `PUBLIC_SITE_URL` was intentionally not used for feed URLs.

## Schema Additions Since The Last Full Docs Sync

- `posts.scheduled_at`
- `posts.pending_platform_ids`
- `posts.featured_image_url`
- `posts.social_post_drafts`
- `platform_connections`
- `post_syndications`
- `platform_oauth_apps`
- `media_assets`
- `art_pieces`
- `art_piece_versions`
- `exhibits`
- `piece_exhibits`
- `media_asset_exhibits`

## Existing Guarantees Reconfirmed

- MySQL remains the canonical datastore.
- Owner-only publishing remains the permission boundary for canonical posts.
- Public exports still exclude non-published posts.
- `GET /export.json`, `GET /feed.xml`, and `GET /feed.json` remain functional.
- Auth.js remains mounted under `/api/auth`.

## Known Documentation Caveats

- This audit treats uncommitted immersive changes as current working-tree state, not as shipped production behavior.
- It does not verify live provider credentials or external API availability.
- It does not mark any AI vendor verified beyond what the existing runbook requires.
