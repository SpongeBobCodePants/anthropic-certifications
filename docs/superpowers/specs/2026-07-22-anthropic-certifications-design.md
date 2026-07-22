# Anthropic Certifications Aggregator — Design Spec

**Date:** 2026-07-22
**Status:** Approved (design), pending spec review

## Overview

A content-driven web app that aggregates information to help people prepare for
Anthropic (Claude) certifications. It serves two purposes:

1. **Directory of info** — an authoritative, easy-to-scan reference for each
   certification: what it is, exam format, cost, domains covered, prerequisites,
   and official links.
2. **Curated community content** — hand-picked tips, study guides, and exam
   experiences gathered from around the web and published as articles.

All content is **curated by the maintainer** and stored as files in the repo.
There is **no database, no user accounts, and no user-submitted content** in v1.

The site launches with a single certification — **Claude Certified Architect –
Foundations** — and is designed so additional certifications can be added by
dropping in new data/content files.

## Goals

- Give exam-takers a single, trustworthy place to understand each certification.
- Make curated tips and study guides easy to browse, filtered by certification.
- Be fast, cheap to run, and simple to maintain (static output, no backend).
- Make adding a new certification a matter of adding files, not writing code.

## Non-Goals (v1 — YAGNI)

- User accounts, login, or authentication.
- User-submitted tips, comments, ratings, or forums.
- A backend database or server-side application logic.
- A CMS / admin UI (content is edited as files via git).
- Automated scraping or auto-aggregation of external sources.
- Practice quizzes / interactive exams (possible future phase, not now).

These are explicitly deferred. The architecture leaves the door open (see
Extensibility) but none are built in v1.

## Users

- **Prospective exam-takers** who want to understand what a certification
  involves before committing.
- **Active studiers** looking for curated guides, tips, and firsthand exam
  experiences.

The site is **unofficial** and will state so clearly (see `/about`), directing
users to official Anthropic resources for authoritative details and registration.

## Architecture

A **SvelteKit** application, **fully prerendered to static HTML** at build time
and deployed on **Vercel**.

- **Framework:** SvelteKit (Svelte 5), TypeScript throughout.
- **UI:** shadcn-svelte components on top of Tailwind CSS (Tailwind is a required
  dependency of shadcn-svelte). Light/dark mode via shadcn theming (`mode-watcher`).
- **Content authoring:**
  - **Certifications** → typed TypeScript data modules (structured metadata).
  - **Guides** (curated tips / study guides / experiences) → Markdown files with
    frontmatter, compiled via **mdsvex**.
- **Rendering:** every route is prerendered (`export const prerender = true` in
  the root layout). Output is static; no serverless functions required for v1.
- **Adapter:** `@sveltejs/adapter-vercel`. Chosen over `adapter-static` because
  it produces the same static output on Vercel while leaving room to add a single
  server route later (e.g. a contact form) without re-architecting.

### Why this approach

Alternatives considered:

- **adapter-static (pure SSG):** simplest, but forecloses any future server
  route. Rejected only because adapter-vercel gives the same result with more
  headroom at no extra cost.
- **On-demand SSR:** unnecessary — there is no dynamic or per-request data.
  More cost and complexity for zero benefit. Rejected.
- **adapter-vercel + global prerender (chosen):** static-fast output, trivial
  Vercel integration, future-proof.

## Content Model

### Certifications (structured data)

Each certification is a typed object. Stored one-per-file under
`src/lib/content/certifications/<slug>.ts`, aggregated by an `index.ts` that
exports a typed array. Adding a cert = add a file + register it in the index.

```ts
// src/lib/content/certifications/types.ts
export interface CertificationDomain {
  name: string;          // e.g. "Agentic Architecture"
  description?: string;   // one-line summary of what the domain covers
  weight?: number;        // optional % of exam, if published
}

export interface Certification {
  slug: string;                 // URL slug, e.g. "claude-certified-architect-foundations"
  name: string;                 // full display name
  shortName: string;            // e.g. "CCA-F"
  role: 'Practitioner' | 'Architect' | 'Developer';
  level: string;                // e.g. "Foundations"
  status: 'available' | 'coming-soon';
  summary: string;              // 1-2 sentence blurb for cards/lists
  description: string;          // longer prose for the detail page (plain text/markdown-lite)
  technologies: string[];       // e.g. ["Claude Code", "Claude Agent SDK", "Claude API", "MCP"]
  domains: CertificationDomain[];
  format: {
    questionCount: number;      // e.g. 60
    durationMinutes: number;    // e.g. 120
    deliveryModes: string[];    // e.g. ["Online proctored", "Test center"]
    questionType: string;       // e.g. "Multiple choice"
  };
  passingScore: { scaled: number; of: number }; // e.g. { scaled: 720, of: 1000 }
  cost: { amount: number; currency: string };   // e.g. { amount: 125, currency: "USD" }
  validityMonths: number;       // e.g. 12
  prerequisites: string[];      // free-text list; empty array if none
  officialUrl: string;          // canonical Anthropic Academy page
  registrationUrl: string;      // where to register/sit the exam
  lastUpdated: string;          // ISO date the maintainer last verified the data
}
```

The cert detail page renders these fields with shadcn components (cards, badges,
a domains table, a "quick facts" panel). Because the data is structured (not
freeform markdown), the presentation is consistent across every certification.

### Guides (Markdown + frontmatter)

Curated tips, study guides, and exam experiences. Stored as `.md` under
`src/content/guides/`, compiled by mdsvex, loaded via `import.meta.glob` at build.

```yaml
---
title: "How I passed CCA-F in two weeks"
description: "A concise study plan focused on the five exam domains."
certSlug: "claude-certified-architect-foundations"  # or "general"
category: "exam-experience"   # study-guide | tips | exam-experience | resource
tags: ["study-plan", "mcp", "agent-sdk"]
author: "Jane Doe"            # optional; original author when aggregated
sourceUrl: "https://..."      # optional; attribution link when content is aggregated
publishedDate: "2026-07-20"
updatedDate: "2026-07-21"     # optional
---

Markdown body...
```

`slug` is derived from the filename. `category` drives grouping and filtering.
`certSlug` links a guide to its certification (`"general"` for cross-cutting
content). When content is aggregated from elsewhere, `sourceUrl`/`author` provide
attribution.

## Routes

| Route | Purpose | Rendering |
|-------|---------|-----------|
| `/` | Landing: what the site is, list of certifications, featured guides | prerendered |
| `/certifications` | Directory of all certifications, client-side filterable | prerendered |
| `/certifications/[slug]` | One certification: full details + related guides | prerendered (one page per cert) |
| `/guides` | Browse all guides, filterable by certification and category | prerendered |
| `/guides/[slug]` | A single guide article (rendered Markdown) | prerendered (one page per guide) |
| `/about` | What the site is, unofficial disclaimer, links to official resources | prerendered |

Dynamic routes (`[slug]`) use `entries()` to enumerate all slugs at build so
every page is prerendered to static HTML.

## Data Flow

1. Certification data and guide Markdown live in the repo as files.
2. `load` functions (in `+page.ts` / `+layout.ts`) import the typed cert data and
   glob-import guides at build time.
3. `entries()` on dynamic routes enumerates every slug so all pages prerender.
4. SvelteKit prerenders all routes to static HTML/CSS/JS.
5. Vercel serves the static output from its edge/CDN.

No runtime data fetching; everything is resolved at build.

## Search & Filtering

- **Certifications directory:** client-side filtering by `role`, `status`, and
  `technology`. Simple, synchronous, over the in-memory cert array.
- **Guides:** client-side filtering by `certSlug` and `category`, plus tag chips.
- **Full-text search:** out of scope for v1. If desired later, a small
  client-side fuzzy index (e.g. Fuse.js over a prebuilt JSON) can be added
  without server changes. Noted, not built.

## UI / Styling

- **shadcn-svelte** for components: `card`, `badge`, `button`, `input`,
  `select`, `separator`, `table`, `tabs`, navigation, etc. Components are
  generated into `src/lib/components/ui/` via the shadcn CLI (copied in, owned by
  the project).
- **Tailwind CSS** for layout and utilities.
- **Dark/light mode** via `mode-watcher`, respecting system preference with a
  manual toggle in the header.
- A shared app shell: header with nav + theme toggle, a footer with the
  "unofficial" disclaimer and a link to Anthropic's official pages.
- Responsive by default; content-first, readable typography for guide articles.

## Testing

Light-touch, matched to a mostly-static content site:

- **Vitest (unit):** content helpers — cert lookup by slug, guide frontmatter
  parsing/normalization, filtering logic, slug derivation. These are the parts
  with real logic and the most likely to break when adding content.
- **Playwright (smoke, a few tests):** home renders and lists certs; a cert
  detail page renders its quick-facts and related guides; a guide article
  renders; navigation between key pages works; theme toggle works.

Follow test-driven development for the content/filter helpers (real logic).
Page-level smoke tests are written alongside the pages.

## Deployment

- Adapter: `@sveltejs/adapter-vercel`.
- The repo connects to the **existing GitHub repository**; the **existing Vercel
  project** is linked to that repo.
- Push to `main` → Vercel builds (`vite build`) and deploys the prerendered
  output automatically.
- Node version pinned via `.nvmrc`/`package.json` `engines` to match Vercel's
  supported runtime.
- `.env.example` documents any config (none required for v1).

## Extensibility (adding certifications / future phases)

- **New certification:** add `src/lib/content/certifications/<slug>.ts` and
  register it in the index. Its directory entry, detail page, and guide filtering
  all work automatically. `status: "coming-soon"` supports announcing a cert
  before its data is complete.
- **New guide:** drop a `.md` file into `src/content/guides/`. It appears in
  `/guides`, filtered to its `certSlug` and `category`, and links from the
  related cert page.
- **Future phases (not built now):** full-text search (client-side index),
  practice quizzes, and a user-submissions flow (would introduce a database and
  auth) are all compatible with this structure but explicitly out of scope for v1.

## Seed Content: Claude Certified Architect – Foundations

The launch certification. Verified details (as of 2026-07-22; maintainer to
re-verify against official sources before publishing):

- **Name:** Claude Certified Architect – Foundations
- **Short name:** CCA-F
- **Role:** Architect · **Level:** Foundations
- **Summary:** Validates that practitioners can make informed tradeoff decisions
  when building real-world solutions with Claude.
- **Technologies covered:** Claude Code, Claude Agent SDK, Claude API, Model
  Context Protocol (MCP).
- **Domains (5):** Agentic Architecture; Tool Design & MCP; Claude Code
  Configuration; Prompt Engineering; Context Management.
- **Format:** 60 multiple-choice questions, 120 minutes.
- **Passing score:** 720 / 1000.
- **Cost:** $125 USD.
- **Delivery:** Online proctored or at a test center (Pearson VUE).
- **Validity:** 12 months.
- **Prerequisites:** None stated.
- **Official page:** https://anthropic.skilljar.com/claude-certified-architect-foundations-certification

Sources consulted:
- Anthropic Academy — Claude Certified Architect – Foundations
  (anthropic.skilljar.com)
- Anthropic Claude Certification Program — Pearson VUE
  (pearsonvue.com/us/en/anthropic.html)

> Note: exam details are set by Anthropic and can change. All published figures
> carry a `lastUpdated` date and the site states it is unofficial.

## Open Decisions (defaults chosen; easy to change)

1. **Curated-content section name:** defaulting to **"Guides"** (`/guides`), with
   a `category` field distinguishing study-guides, tips, exam-experiences, and
   resources. Can be renamed (e.g. "Tips") trivially.
2. **GitHub remote URL:** needed to wire up the remote and first push — to be
   provided by the user before scaffolding pushes anything.

## Milestones (high level; detailed plan comes next)

1. Scaffold SvelteKit + TypeScript + adapter-vercel; init Tailwind + shadcn-svelte.
2. Wire git remote to the existing GitHub repo; confirm Vercel builds a hello page.
3. Content layer: cert types, seed CCA-F data, mdsvex + guide loader.
4. Pages: app shell, home, certifications directory + detail, guides list + article, about.
5. Filtering, theming, polish; tests; final deploy.
