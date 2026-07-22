# CLAUDE.md

Guidance for working in this repository.

## What this is

A content-driven web app that aggregates information to help people prepare for
**Anthropic (Claude) certifications** — an authoritative directory of each
certification plus curated tips, study guides, and exam experiences. All content
is curated (files in the repo); there is **no database, no auth, no user
accounts, and no user-submitted content**.

Launch certification: **Claude Certified Architect – Foundations (CCA-F)**. The
app is designed so additional certifications can be added by dropping in files.

The full product design lives in
[docs/superpowers/specs/2026-07-22-anthropic-certifications-design.md](docs/superpowers/specs/2026-07-22-anthropic-certifications-design.md).
Note: the app UI/content has **not** been designed or built yet — the project is
currently at the "scaffold complete" stage.

## Tech stack

- **SvelteKit** (Svelte 5, **runes mode is forced** for project code — see
  `vite.config.ts`) + **TypeScript**.
- **Tailwind CSS v4** (via `@tailwindcss/vite`) + **shadcn-svelte** for components.
- **`@sveltejs/adapter-vercel`**, deployed on **Vercel**.
- Tooling: Prettier, ESLint, Vitest (unit).

Note: this scaffold **inlines the SvelteKit config into `vite.config.ts`**
(the `sveltekit()` plugin receives `compilerOptions` and `adapter`). There is
**no `svelte.config.js`** — that is intentional for this template version, not
missing.

## Commands

```sh
npm run dev        # start dev server (use this for local development)
npm run build      # production build (see Windows caveat below)
npm run preview    # preview a production build
npm run check      # svelte-kit sync + svelte-check (type checking)
npm run lint       # prettier --check + eslint
npm run format     # prettier --write
npm run test       # run unit tests once (vitest)
npm run test:unit  # vitest (watch)
```

### Windows local-build caveat

`npm run build` runs the Vercel adapter, which creates **symlinks**. On Windows
without Developer Mode / admin rights this fails with `EPERM ... symlink`. This
is a **local-only** limitation — **Vercel's own builds run on Linux and are
unaffected**. For local work use `npm run dev` (no adapter) and `npm run check`
to verify. To make `npm run build` succeed locally, enable Windows Developer Mode
(Settings → Privacy & security → For developers) or run the shell as admin.

## Project structure

```
src/
  app.html, app.d.ts
  routes/
    +layout.svelte        # imports ./layout.css
    layout.css            # global CSS: Tailwind + shadcn theme tokens (neutral base)
    +page.svelte
  lib/
    components/ui/         # shadcn-svelte components land here (owned by us)
    hooks/                 # shadcn-svelte hooks
    utils.ts              # `cn()` class-merge helper + shadcn type helpers
    assets/
static/                    # static assets served at /
docs/superpowers/specs/    # design specs
```

Path aliases (from `components.json` / SvelteKit): `$lib`, `$lib/components`,
`$lib/components/ui`, `$lib/utils`, `$lib/hooks`.

## shadcn-svelte

- Config: `components.json` (style `nova`, base color `neutral`, icons `lucide`).
- Add components with the CLI, e.g.:
  ```sh
  npx shadcn-svelte@latest add button card badge input
  ```
  Components are copied into `src/lib/components/ui/` and are ours to edit.
- **Non-interactive note (this environment):** the shadcn-svelte CLI is
  interactive and cancels on EOF in a non-TTY shell. Pipe confirmations with
  `yes | npx shadcn-svelte@latest ...` to auto-accept prompts. `init` also needs
  a `--preset` code; the all-defaults preset (neutral base, `nova` style) is
  `b0`.
- Theme tokens live in `src/routes/layout.css`; dark mode uses the `.dark` class
  (`@custom-variant dark`).

## Conventions

- Use Svelte 5 runes (`$props`, `$state`, `$derived`, …); the compiler enforces
  runes mode for project code.
- Use the `cn()` helper from `$lib/utils` for conditional class names.
- Prefer prerendering (static output) — content is known at build time. There is
  no per-request server data in v1.

## Deployment

- Connected to a GitHub repo; the existing Vercel project auto-builds on push to
  `main`. Vercel runs `vite build` with the Vercel adapter and serves the output.
