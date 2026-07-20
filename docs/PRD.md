# Build Plan — Security News Blog (Astro + Strapi + Neon)

Planning document for a small, content-centered publication site demonstrating a full headless CMS pipeline: non-technical editors manage all content in a CMS; the public site is fully static and rebuilds automatically on publish.

Reference guide for the Strapi + Neon plumbing: https://neon.com/guides/strapi-cms

---

## Goal

A live security-news mini publication (short posts about AI security topics) where:

- the public site is static, fast, and accessible (Astro + Tailwind + Preline)
- all content is authored in a headless CMS (Strapi) by non-technical editors
- data persists in managed Postgres (Neon)
- publishing content triggers an automatic site rebuild (webhook → deploy hook)

## Architecture

```
[Editor] → Strapi admin (Render free tier) → Neon Postgres (persistent, free tier)
                    │ publish/unpublish webhook
                    ▼
            Vercel deploy hook → Astro rebuild (~30-60s) → static site (Vercel free tier)
```

Key properties:

- **Build-time data fetching (SSG).** Astro fetches Strapi's REST API at build time only. The public site is pure static HTML — CMS availability and cold starts never affect visitors.
- **Persistent external database.** Strapi's default SQLite lives on the local filesystem; on hosts with ephemeral disks (Render free tier) it is wiped on every restart. Pointing Strapi at Neon Postgres via `DATABASE_URL` makes the data survive restarts and redeploys.
- **Cold starts are acceptable.** Render free tier sleeps the Strapi service after ~15 min idle (~30–60s wake). This only affects the admin panel, never the public site.

## Decisions and rationale

| Decision                             | Rationale                                                                                                                                                                            |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Astro over Next.js                   | Content site: static output, zero client JS by default, content-driven routing (`getStaticPaths`) is the core idiom                                                                  |
| Strapi (self-hosted) over hosted CMS | TypeScript-based, auto-generated REST API, schema lives in git; exercises the full stack end to end                                                                                  |
| Neon Postgres over default SQLite    | SQLite on ephemeral disk = data loss on restart; external managed Postgres persists                                                                                                  |
| No ORM, no hand-written migrations   | Strapi owns the schema: content types are git-versioned `schema.json` files; Strapi auto-migrates the DB on boot. The site talks to Strapi's REST API only, never to the DB directly |
| No media uploads                     | Uploaded files land on the ephemeral disk and vanish; text-only fields (external image URLs if ever needed)                                                                          |
| Plain monorepo, no workspace tooling | The two apps share zero code; separate `package.json` per folder is simpler than workspaces/Turborepo                                                                                |
| SSG over SSR                         | Editors publish rarely; visitors read often. Rebuild-on-publish gives static speed with CMS freshness                                                                                |

## Repo layout

```
security-news-blog/
  cms/     # Strapi (own package.json)
  web/     # Astro  (own package.json)
  README.md
```

- `.gitignore` per folder: `node_modules`, `.env`, `cms/.tmp/`
- Secrets (Neon connection string, API tokens) live in platform env vars; `.env.example` documents required keys
- Deploy config: Vercel Root Directory = `web/`; Render Root Directory = `cms/`

## Content model

1. **Homepage** — Single Type: `title`, `tagline`
2. **Post** — Collection Type: `title`, `slug` (UID type — auto-generated from title, unique, editors never think about URLs), `excerpt`, `body` (rich text)

No custom date/author fields — Strapi provides them as system fields:

- `publishedAt` — set automatically by Draft & Publish on publish
- `updatedAt` — serves as **lastEditedAt**; bumped on every save, by any editor
- `createdBy` — the admin user who originally created the entry (auto-populated from the logged-in account, no manual input). Not exposed over REST by default; enable with `"options": { "populateCreatorFields": true }` in the Post `schema.json`, then fetch with `populate=createdBy` (returns firstname/lastname, not email)

Any admin user (Super Admin or Editor) can edit any post; `updatedAt`/`updatedBy` track the last edit automatically.

Seed posts (short, ~150 words each):

- "What is prompt injection"
- "Why this site runs on a headless CMS"
- "RAG in two paragraphs"

## Site structure (2 routes, both static)

```
web/src/pages/
  index.astro         # landing: masthead + post list (newest first) + footer
  posts/[slug].astro  # getStaticPaths() → one prebuilt page per post
```

`[slug].astro` is statically generated per post at build time — not runtime-dynamic. New posts get their page on the publish-triggered rebuild.

### UI

- Tailwind via `npx astro add tailwind`; Preline static blocks (masthead, single-column blog list, footer) copy-pasted and wired to CMS data
- No Preline JS plugin (only needed for interactive components — none here)
- Single column, `max-w-2xl mx-auto`, semantic HTML with real heading hierarchy
- Footer: "Built with Astro · Strapi · Neon — content managed headlessly, site rebuilds on publish" + "Developed by Salim Dellali" with a link to the GitHub repo

## Build order (thin end-to-end slice first, then widen)

1. Neon: create project, copy connection string (`sslmode=require&channel_binding=require`)
2. `npx create-strapi-app` in `cms/` — postgres client, SSL enabled; `Post` with 2 fields; 1 entry
3. `curl` the REST API locally — verify JSON
4. Astro in `web/`: simple nav bar (no links, just "Security News Blog" centered) + a bare `<ul>` of posts to verify linking works — fancy cards come later (step 6)
5. **Deploy both (Render + Vercel) — same list live ← the milestone that matters**
6. Widen: full `Post` fields, `Homepage` single type, `posts/[slug].astro`, seed content, Preline styling
7. Webhook: Strapi Settings → Webhooks → publish/unpublish events → Vercel Deploy Hook URL
8. README with architecture diagram

API access: Settings → Roles → Public → enable `find`/`findOne` on Post + Homepage (or a read-only API token in the build environment).

## Editorial roles

- **Super Admin** — owner (first registered account, mine): schema, settings, users
- **Editor** — content contributors (TrendAI account): create/edit/publish any entry; no access to schema, settings, tokens, or users

Both accounts can edit any post; Strapi's `createdBy`/`updatedBy` attribute authorship and last-edit to the right account automatically.

This split mirrors a real newsroom/marketing setup: developers own the model, editors own the words.

## Fallback if the evening derails

Deployed Astro site + locally-run Strapi still demonstrates the full pipeline. Cut the Homepage type and styling polish before cutting the deployed public site.
