# SEO Architecture Plan — Next.js + Payload CMS

## Your task

Design a complete, production-ready SEO architecture for a Next.js 14+ (App Router)
project using Payload CMS v3 as the headless backend. The output must be an
**architectural plan** — a structured document listing every file to create,
every field to add, and every integration to wire up. Do NOT write full
implementation code unless a specific snippet is critical for understanding.

---

## Project context (fill in before using this prompt)

- **Project name**: [YOUR PROJECT NAME]
- **Domain**: [https://yourdomain.com]
- **Languages / locales**: [e.g. en (default), uk — or just en]
- **Collections** (content types that need per-page SEO):
  [List each collection, e.g.:
    - Pages (static landing pages)
    - Posts (blog articles)
    - Products (e-commerce items)
    - Categories (taxonomy pages)
    - ... add as many as needed]
- **Globals** (site-wide CMS sections, NOT needing per-page SEO):
  [e.g. Navbar, Footer, Hero, SiteSettings]
- **Schema.org types relevant to this project**:
  [e.g. Organization, WebSite, JobPosting, Product, Article, BreadcrumbList,
   FAQPage — pick what fits the domain]
- **AI provider for SEO generation**: Anthropic Claude (model: claude-haiku-4-5-20251001)
- **Analytics tools expected**: [e.g. Google Analytics 4, Google Tag Manager, or none]

---

## Required SEO features — plan ALL of the following

### 1. SeoGlobal (Payload Global)

Design a Payload Global with slug `seo-settings`, grouped under "Settings" in the
admin sidebar. Use tabs to organize fields. Required tabs:

- **Indexing & Robots**
  - Global site indexing toggle (checkbox) — controls robots.txt
  - Custom robots rules (array): user-agent, allow paths[], disallow paths[]

- **Sitemap**
  - Enable/disable sitemap.xml (checkbox)
  - Exclude slugs from sitemap (array of text fields)

- **Redirects**
  - Redirect rules (array): from path, to destination, permanent (301 vs 302)
  - Note: these must work via Next.js middleware, no redeploy required

- **Default SEO**
  - Title template with `%s` placeholder (e.g. `%s | Brand Name`)
  - Default meta title — one field per locale
  - Default meta description — one field per locale
  - Default OG image (upload, relation to media collection)

- **Verification**
  - Google Search Console verification token
  - Bing Webmaster Tools verification token

- **Analytics**
  - Google Analytics 4 Measurement ID (`G-XXXXXXXXXX`)
  - Google Tag Manager Container ID (`GTM-XXXXXXX`)
  - Note: if GTM is set, GA4 must NOT inject its own script (avoid double tracking)

- **Social**
  - Twitter/X handle
  - Facebook page URL
  - LinkedIn URL

---

### 2. Reusable SEO field group (shared across collections)

Design a reusable `seoFields` array (exported from `fields/seo.ts`) to be
embedded as a "SEO" tab in every collection listed in the Project context above.

Required fields inside the group (name: `seo`):

- `metaTitle` (text, EN) — hint: 50–60 chars
- `metaDescription` (textarea, EN) — hint: 150–160 chars
- One `metaTitle` and one `metaDescription` per additional locale
- `ogImage` (upload, relation to media)
- `noIndex` (checkbox, default false) — per-page override
- `aiSeoButton` (ui field) — renders `AiSeoButton` client component
- `seoPreview` (ui field) — renders `SeoPreview` client component

Specify exactly how this group is added to each collection (as a tab, not inline).

---

### 3. Admin UI components

#### AiSeoButton (`components/admin/AiSeoButton.tsx`)

Plan a Payload client component that:
- Reads the document's title fields (EN + all other locales) from the live form
  via `useFormFields`
- On click, calls `POST /api/generate-seo` with `{ title, titleUk, collection }`
- On success, dispatches `UPDATE` actions to fill `seo.metaTitle`,
  `seo.metaDescription`, and locale variants via `dispatchFields`
- Shows three states visually: idle / loading / success / error (auto-resets after 3s)
- Is protected: the API route must require an active Payload admin session

#### SeoPreview (`components/admin/SeoPreview.tsx`)

Plan a Payload client component that:
- Reads `seo.metaTitle`, `seo.metaDescription`, `seo.ogImage` live via `useFormFields`
- Renders three preview tabs: **Google Search**, **Facebook / OG**, **X / Twitter**
- Google tab shows: favicon placeholder, domain, title (truncated at 60), description
  (truncated at 160), and character count indicators (green/orange/red thresholds)
- Facebook tab shows: OG image at 1.91:1 ratio, domain, title, description
- Twitter tab shows: large image card layout
- Falls back gracefully when fields are empty

---

### 4. AI SEO generation API (`app/api/generate-seo/route.ts`)

Plan a `POST` handler that:
- Requires authenticated Payload session (reject 401 if no user)
- Accepts `{ title: string, titleUk?: string, collection: string }`
- Calls `claude-haiku-4-5-20251001` via `@anthropic-ai/sdk`, max 400 tokens
- Prompt must ask for valid JSON only with keys:
  `metaTitle`, `metaDescription`, plus one pair per locale beyond EN
- Strips markdown code fences from response before `JSON.parse`
- Returns the parsed object or `{ error, raw }` with 500 on parse failure
- Must NOT expose `ANTHROPIC_API_KEY` to the client

---

### 5. Next.js metadata — layouts

For each locale, plan a `generateMetadata()` async function in its route layout:

- Calls `getSeoSettings()` (cached with React `cache()`)
- Constructs `Metadata` object with:
  - `metadataBase`
  - `title.default` + `title.template` from CMS
  - `description` from CMS (locale-aware)
  - `keywords` array (hardcoded per project domain — list the keywords)
  - `alternates.canonical` and `alternates.languages` for hreflang
    (one entry per locale + `x-default`)
  - `openGraph` — title, description, url, siteName, locale (`en_US` or `uk_UA`),
    type, images array with width/height
  - `twitter` — `summary_large_image` card, title, description, images,
    site/creator from CMS social handle
  - `verification` — google and bing tokens from CMS (conditional — only if set)
  - `robots.index` and `robots.follow` — driven by `siteIndexing` global flag

For each collection listed above, plan a `generateMetadata()` at its dynamic
route that:
- Fetches the document from Payload by slug
- Merges `seo.metaTitle` / `seo.metaDescription` (locale-aware) on top of globals
- Respects `seo.noIndex` checkbox to override `robots.index`
- Sets `alternates.canonical` to the canonical URL of that document

---

### 6. robots.txt (`app/robots.ts`)

Plan a dynamic `robots()` function that:
- Reads `siteIndexing` + `customRobotRules` from `seo-settings` global
- When indexing is ON: `allow: /`, `disallow: ['/admin/', '/api/']`
- When indexing is OFF: `disallow: /` for all bots
- Appends any custom rules from the CMS array after the main rule
- Always includes `sitemap:` and `host:` directives
- Falls back to `disallow: /` if Payload is unavailable

---

### 7. sitemap.xml (`app/sitemap.ts`)

Plan a dynamic `sitemap()` function that:
- Reads `sitemapEnabled` and `sitemapExcludeSlugs` from CMS
- Returns `[]` if disabled
- Always includes static routes: one URL per locale (priority 1.0 / 0.9)
- For each collection that has public URLs: fetches all `published` documents,
  generates one URL per locale per document with `lastModified`, `changeFrequency`,
  `priority`, and `alternates.languages`
- Skips documents whose slug is in `sitemapExcludeSlugs`
- Logs a console warning if `totalDocs` exceeds the fetch limit
- Returns all routes concatenated in a flat array

Specify priority and changeFrequency values for each collection.

---

### 8. Dynamic OG Image (`app/opengraph-image.tsx`)

Plan a Next.js `ImageResponse` (1200×630, nodejs runtime) that:
- Is NOT static — reads document title and OG image URL from the request context
  (or from CMS for the root route)
- Accepts optional `?title=` and `?imageUrl=` search params for dynamic routes
- Renders: background image layer, dark overlay gradient, headline text,
  supporting sub-text, domain name in footer
- Falls back to a gradient background when no image is provided

Specify how each collection's dynamic route links to its own OG image
(via `openGraph.images` in `generateMetadata`).

---

### 9. Dynamic Structured Data / JSON-LD (`app/structured-data.tsx` or co-located)

Plan where and how JSON-LD is injected. For each Schema.org type listed in the
project context, specify:
- Which route(s) render it
- Which fields come from Payload vs hardcoded
- How it is kept up-to-date without redeploy (i.e. driven by CMS data)

Required schemas for all projects:
- `WebSite` with `SearchAction` (SiteLinksSearchBox) — root route only
- `Organization` — root route; fields: name, url, logo, sameAs, contactPoint, address
- `BreadcrumbList` — every dynamic route

Additional schemas per collection:
- For each collection in the project context, specify the most appropriate
  Schema.org type and the field mapping from Payload fields to schema properties.

---

### 10. Redirects middleware (`middleware.ts`)

Plan a Next.js middleware that:
- Fetches redirect rules from `/api/redirects` (internal route that reads from
  Payload `seo-settings` global)
- Caches result in module-level memory with a 60-second TTL
- Matches exact pathname (no wildcard) — returns 301 or 302 accordingly
- Skips `/api/`, `/admin/`, `/_next/` paths
- Sets `x-pathname` response header for locale detection in layouts
- Matcher config: covers all paths except static assets and favicon

---

### 11. Utility: getSeoSettings (`lib/getSeoSettings.ts`)

Plan a single exported function:
- Uses React `cache()` to deduplicate within a single render pass
- Calls `payload.findGlobal({ slug: 'seo-settings', depth: 1 })`
- Returns `null` on error (never throws)
- Is imported by layouts, robots.ts, sitemap.ts, and opengraph-image routes

---

## Output format

Return the plan as a structured document with these sections:

1. **File tree** — every new file with a one-line description
2. **Payload schema summary** — each Global and the SEO tab in each Collection,
   showing field names and types in a table
3. **Data flow diagram** (described in text/ASCII) — how CMS data reaches
   robots.txt, sitemap, metadata, JSON-LD, and OG image
4. **Per-collection SEO checklist** — for each collection, list exactly what
   SEO fields and generated outputs it contributes to
5. **Missing / to-decide** — things you cannot determine without more project
   context (e.g. Schema.org types for domain-specific entities, exact keywords,
   URL structure for custom collections)

Do NOT write full file contents. Write enough detail that a developer can
implement each piece without guessing at the architecture.
