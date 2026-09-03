# CompAtlas (Yazılım Atlası) — Frontend

A directory of every software company in Turkey's 81 provinces that has a live
website — collected, verified, classified and published as one searchable atlas.

> **No separate backend service.** CompAtlas is a Next.js application on top of
> Supabase (Postgres + RLS + Auth + Storage). Business logic lives in the app's
> server components, route handlers and server actions; the batch data
> collection runs as a standalone CLI. There is no API server to deploy.

## Overview

"Which software companies are in Denizli?" has no good answer in Turkey today.
Google Maps mixes in computer repair shops and phone dealers. Business
directories are pay-to-list, stale, and full of companies that closed years ago.
Nobody publishes what is actually there.

CompAtlas answers it. It sweeps open sources — OpenStreetMap, technopark
rosters, Google Places, open POI datasets, sector lists, job boards — normalizes
and de-duplicates the results, checks that each site is actually alive,
classifies it as genuinely software (versus IT retail or repair), scores the
confidence of every record, and publishes the survivors as **13,858 companies
across all 81 provinces**, with at least one company in every single one.

Around the directory sits the part that makes it self-sustaining: companies
claim their own listing through verified ownership, add job postings, receive
quote requests, get reviews, and see a visibility report on their own site's SEO
and Google Business Profile standing.

## Features

**Directory**
- Company profiles with province, district, categories, site summary and badges
- Browse by province (`/sehir`), by category (`/kategori`), and full-text search
  (`/ara`)
- Programmatic SEO landing pages — province × category combinations, plus
  editorial hubs (`/turkiye-yazilim-firmalari`, `/turkiye-yazilim-sektoru`)
- Methodology and data-source attribution pages, because the data provenance is
  part of the product's credibility
- KVKK data-removal request flow (`/veri-kaldirma`)

**Company self-service**
- Claim a listing (`/sahiplen`) with e-mail verification against the company's
  own domain, corporate verification, claim-attempt logging and reversal
- Self-registration for companies not yet in the index (`/firma-ekle`)
- Company panel (`/panel`): listing management, job postings, quote requests,
  applications, notifications, logo upload
- Job postings with structured application questions, plus a public job board
  (`/ilanlar`)
- Quote requests (`/teklif-al`) routed to matching companies
- Reviews and badges

**Visibility product**
- Google OAuth connection to Search Console and Business Profile
- Technical SEO audit, GEO/local analysis, ranking signals, durability scoring
- Generated recommendations, packaged and priced automatically, with periodic
  refresh

**Admin**
- Company, ownership, user, article and error-report moderation under `/admin`
- Daily maintenance and notification jobs behind route handlers

**Platform**
- Turkish/English via `next-intl`
- Bot policy and `llms.txt` for AI crawlers
- Security headers, JSON-LD structured data, generated sitemap and OG images

## Architecture

A pnpm workspace monorepo. The split exists so the parts with real logic can be
tested without a browser or a database.

```
apps/web        Next.js 16 App Router — the site, admin, panel, route handlers
apps/scraper    Node.js CLI — collection, classification, enrichment pipeline
packages/core   Pure TypeScript engines, no I/O
packages/db     Supabase client factory + repository layer
supabase/       migrations, config.toml, snippets, tests
mcp/oracle-db   MCP server for database access from AI tooling
```

**`packages/core` holds every decision rule as a pure function:**
`normalize-domain`, `slugify`, `dedup`, `dogrulama-skoru` (confidence score),
`yazilim-skoru` (is-this-actually-software score), `kategori-siniflandirma`,
`canlilik-kontrol` (liveness), `kvkk-filtre`, `firma-ad-onarim` (name repair),
`site-meta`, `seo-denetim`, and the `gorunurluk/` visibility engines (technical
SEO, GEO analysis, opportunity detection, scoring, durability, recommendations).
None of it touches the network or the database, so all of it is unit-tested —
the package was built to 100% coverage.

**`packages/db` is the only place that talks to Supabase.** A client factory
(anon vs. service role) plus one repository per aggregate: `firma-repo`,
`il-repo`, `kategori-repo`, `sahiplen-repo`, `oz-kayit-repo`, `ilan-repo`,
`teklif-repo`, `talep-repo`, `yorum-repo`, `yazi-repo`, `olay-repo`,
`gorunurluk-repo`, `admin-repo`. Pages and actions never construct a query.

**`apps/scraper` is the pipeline**, one adapter per source — `osm-adapter`,
`places-adapter`, `overture-adapter`, `fsq-adapter`, `teknokent-adapter` and its
Playwright variant, `itu-ari-adapter`, `bilisim500-adapter`,
`techcareer-adapter`, `eposta-domain-adapter`, plus generic one- and two-stage
directory adapters. Each honours `robots.txt` and a shared rate limiter, and
results flow through `pipeline.ts` → core engines → `supabase-writer.ts`.
Separate jobs handle classification (`siniflandirma-job`), site metadata
(`site-meta-job`) and category sweeps (`kategori-job`).

**Data provenance is a first-class column.** Every company records which source
it came from — the coverage table in [`KAPSAM.md`](../compatlas.com.tr/KAPSAM.md)
breaks all 13,858 records down by province and by source (OSM / Technopark /
Directory / Places / Open-POI / Other), including where the free-tier ceiling
was hit. Being able to say where a record came from is what separates this from
a scraped list.

**Security sits in the database, not the app.** Supabase RLS policies, triggers
and RPCs — 40+ migrations' worth — enforce who may read and write what. The
`SUPABASE_SECRET_KEY` service client is used only where a policy cannot express
the rule.

## Tech Stack

Frontend:
- Next.js 16.2 (App Router, React Server Components), React 19.2, TypeScript 5
- Tailwind CSS v4 (`@tailwindcss/postcss`)
- `next-intl` v4 for TR/EN routing and messages
- `lottie-web` for animation, `@vercel/analytics`
- `vitest` for unit tests, ESLint 9 + `typescript-eslint`

Backend:
- None as a deployable service. Server logic runs inside `apps/web` — server
  components, route handlers under `/api` (`veri`, `olay`, `bakim`,
  `gorunurluk`, `bildirim`), and server actions.
- `@anthropic-ai/sdk` for AI-assisted classification and content
- `resend` for transactional e-mail

Database:
- Supabase — PostgreSQL with Row Level Security, Auth (Google OAuth), Storage
  (company logos), `pg_cron` for scheduled notifications
- Schema as versioned SQL migrations in `supabase/migrations/`

Data pipeline:
- Node.js CLI (`tsx`), `playwright` for JS-rendered sources, `cheerio` for HTML
  parsing, `pino` for structured logs
- Sources: OpenStreetMap, Overture, Foursquare OS Places, Google Places API,
  technopark rosters, sector lists, job boards, e-mail domain records

Infrastructure:
- Vercel — root directory `apps/web`, pnpm workspace resolved from the repo root
- Domain `compatlas.com.tr`, TLS issued by Vercel
- pnpm 11 workspace, Node.js ≥ 20

## Running Locally

**Prerequisites:** Node.js ≥ 20, pnpm 11, Docker Desktop (for the local Supabase
stack), Supabase CLI (`npx supabase@latest` works — no global install needed).

1. **Install**

   ```bash
   pnpm install
   ```

2. **Environment.** Copy `.env.example` to `apps/web/.env.local`:

   | Variable | Used by | Required |
   |---|---|---|
   | `NEXT_PUBLIC_SUPABASE_URL` | web, db | yes |
   | `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | web, db (anon client) | yes |
   | `NEXT_PUBLIC_SITE_URL` | canonical URLs, OG, sitemap | yes |
   | `SUPABASE_SECRET_KEY` | scraper, service client, admin RPCs | for admin/claim flows |
   | `PLACES_API_KEY` | scraper (Google Places adapter) | for scraping |
   | `RESEND_API_KEY` | `/api/hooks/email` | for e-mail |
   | `ANTHROPIC_API_KEY` | AI classification and content | for those features |

3. **Bring up the database**

   ```bash
   npx supabase@latest start      # local Postgres + Studio in Docker
   npx supabase@latest db reset   # apply all migrations + seed
   ```

   Or point at the hosted project and push:

   ```bash
   npx supabase@latest link --project-ref <ref>
   npx supabase@latest db push
   ```

4. **Run the site**

   ```bash
   pnpm dev          # http://localhost:3000
   ```

5. **Run the pipeline**

   ```bash
   pnpm --filter @yazilim-atlasi/scraper start        # collection
   pnpm --filter @yazilim-atlasi/scraper siniflandir  # classification
   ```

   The scraper needs `SUPABASE_SECRET_KEY` and the relevant source API keys in
   `apps/scraper/.env.local`.

6. **Verify**

   ```bash
   pnpm build       # all workspace packages
   pnpm lint
   pnpm test        # vitest across the workspace
   ```

   To build while `next dev` is running, set `NEXT_DIST_DIR` so the build does
   not overwrite the dev server's `.next`:

   ```bash
   NEXT_DIST_DIR=.next-verify pnpm --filter yazilim-atlasi build
   ```

## Production

**Vercel**, with the repository imported directly:

| Setting | Value |
|---|---|
| Root Directory | `apps/web` |
| Framework | Next.js (auto-detected) |
| Build Command | default `next build` |
| Install Command | default `pnpm install` (lockfile read from repo root) |
| Node.js | 20.x or newer |

No `vercel.json` — security headers are declared in `apps/web/next.config.ts`
(`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
`Permissions-Policy`, HSTS with preload). A CSP is deliberately not set yet: it
would break the inline theme script and `next/og`, and is scheduled as a v2
hardening step.

**Domain.** `compatlas.com.tr` and `www.` on Vercel; apex via A record
`76.76.21.21` or ALIAS to `cname.vercel-dns.com`, `www` via CNAME. Certificates
are issued automatically.

**Environment variables** are set per-environment in the Vercel dashboard.
`NEXT_PUBLIC_*` values are inlined at build time — changing one requires a
redeploy.

**Database** is the hosted Supabase project. Schema changes ship as migrations
(`supabase db push`), never as dashboard edits, so the local stack and
production stay identical. `pg_cron` drives scheduled notifications inside the
database rather than from the web tier.

**The scraper does not run on Vercel.** It is a long-running Playwright process
against rate-limited third-party sources — run on demand from a workstation or a
scheduled runner, writing through the service-role client.

**Post-deploy checks** (from [`docs/DEPLOY.md`](../compatlas.com.tr/docs/DEPLOY.md)):
`/`, `/robots.txt`, `/sitemap.xml`, `/manifest.webmanifest`, `/opengraph-image`,
a company page's JSON-LD, and `curl -I` for the security headers.

## Technical Decisions

**Why Supabase instead of a Django/ASP.NET backend?**
Almost every write in this product is a permission question rather than a
computation: may this user edit this company, may this claim be approved, may
this review be published. Those rules are best expressed as RLS policies next to
the data, where nothing can bypass them — not in a service layer that every new
endpoint has to remember to call. Supabase also supplies Auth, Storage and
`pg_cron`, which removes three components that would otherwise need running. The
logic that *is* real computation — scoring, classification, deduplication — lives
in `packages/core` where it is testable, not in a controller.

**Why PostgreSQL?**
Full-text search across 13k companies, geographic filtering by province and
district, category many-to-many joins, and hierarchical province/district data —
all in one engine, all indexable. The directory queries are joins across five or
six tables; that is the shape Postgres is best at. Just as decisive: RLS. A
document store has no equivalent for "this row is readable by everyone but
writable only by its verified owner."

**Why no Redis?**
The directory is read-heavy and almost entirely static between scraper runs, so
the caching that matters happens above the database, not beside it — Next.js
static generation and ISR serve most pages from the edge without touching
Postgres at all. A Redis layer would sit between the app and a database the app
mostly is not calling. `lib/onbellek.ts` handles the small amount of in-process
caching that remains.

**Why no Celery-style job queue?**
The batch work is a scraper that runs on demand and takes hours — that is a CLI,
not a queue task. The recurring work is notifications, which run in the database
on `pg_cron`, and daily maintenance, which is a route handler hit by a
scheduler. Neither needs a broker, a worker fleet, or a result backend.

**Why no Elasticsearch?**
Postgres full-text search is enough for 13,858 rows across a handful of fields.
An external index would need reindexing on every scraper write, claim, edit and
KVKK removal — a second consistency problem in exchange for latency nobody is
currently feeling. If volume grows an order of magnitude, or search moves to
fuzzy/semantic matching, this is the decision to revisit.

**Why a monorepo with a pure-TypeScript core?**
The same rules run in two places — the scraper applies them at ingest, the web
app applies them when a company self-registers or edits. Duplicating "is this
domain the same as that one" or "is this actually a software company" in two
codebases guarantees drift. Keeping them in a dependency-free package means one
implementation and a test suite that needs neither a database nor a browser;
`packages/core` was driven to 100% coverage precisely because that is achievable
when a package has no I/O.

**Why consume workspace packages as TypeScript source?**
`packages/core` and `packages/db` expose `./src/index.ts` directly rather than a
build artifact, so there is no build step between editing a rule and seeing it
in the site. The cost is `transpilePackages` in `next.config.ts` — Next has to
transpile them itself to resolve NodeNext-style `.js` imports. That is one
config line versus a watch-and-rebuild loop in every dev session.

**Why `next-intl` here, when the sister project hand-rolls i18n?**
Because this site has thousands of generated routes — province, category,
company, article — and each needs a locale-correct URL, canonical tag and
sitemap entry. That is routing-level localization, which is exactly what
`next-intl` provides and what a Context-based `t()` helper cannot.

**Why Vercel rather than the Kubernetes cluster the other projects use?**
This app's performance profile is ISR and edge-cached static pages across a
large generated route set. That is Vercel's native model, and it comes with no
cluster, no ingress, and no certificate management. There is no long-running
process to host — the only one, the scraper, deliberately runs elsewhere.

**Why record the data source on every company?**
The product's claim is accuracy, and accuracy is unverifiable without
provenance. Recording the source per record makes coverage auditable
(`KAPSAM.md`), makes attribution possible (`VERI-KAYNAKLARI-ATIF.md`), and makes
a bad source removable in one query instead of a re-scrape.

## Screenshots

Design references live in `docs/design/`.

| Page | File |
|---|---|
| Home — map and counters | `docs/screenshots/anasayfa.png` |
| Province page | `docs/screenshots/sehir.png` |
| Company profile | `docs/screenshots/firma.png` |
| Search | `docs/screenshots/ara.png` |
| Claim flow | `docs/screenshots/sahiplen.png` |
| Company panel | `docs/screenshots/panel.png` |
| Visibility report | `docs/screenshots/gorunurluk.png` |
| Admin | `docs/screenshots/admin.png` |

Live: <https://compatlas.com.tr>

## Roadmap

From `docs/PLAN-production-hazirlik.md`, `docs/GROWTH_MASTER_PLAN.md` and
`docs/SONRAKI-ADIMLAR-seo.md`:

- [ ] Content Security Policy — deferred at v1 because of the inline theme
      script and `next/og`
- [ ] District-level depth for the provinces still covered only at city level
      (`PLAN-firma-ilce-derinlik.md`)
- [ ] Re-scrape the provinces capped by free-tier source limits
      (`PLAN-kalan-iller-firma-tarama.md`)
- [ ] Periodic liveness re-checks so closed companies age out automatically
- [ ] SEO/GEO phases A–B rollout (`RUNBOOK-seo-geo-faz-ab.md`)
- [ ] Paid visibility packages and subscription billing
- [ ] Scheduled scraper runs instead of manual invocation
- [ ] Public API for the directory
- [ ] Raise `apps/web` test coverage toward what `packages/core` already has

## License

Proprietary — CompAtlas / Yazılım Atlası. All rights reserved.
