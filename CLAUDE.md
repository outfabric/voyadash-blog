# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Package manager: **pnpm** (Node 20.18.0 via `.nvmrc`).

- `pnpm dev` — runs Tina CMS wrapping `astro dev` (CMS admin at `/admin`). Use this when CMS-edited content matters.
- `pnpm start` — `astro dev` only, no Tina wrapper. Faster startup when editing code/templates.
- `pnpm build` — `astro build` followed by `pagefind --site dist` (`postbuild`) to generate the static search index. Pagefind only works against the built `dist/`, so search will be missing in dev.
- `pnpm preview` — preview the production build locally.
- `pnpm sync` — regenerate `astro:content` types after changing `src/content/config.ts` or the content collection schema.
- `pnpm lint` — ESLint (astro plugin + `@typescript-eslint/parser`).
- `pnpm format` / `pnpm format:check` — Prettier with `prettier-plugin-astro`.

Husky `pre-commit` runs `lint-staged`, which formats staged `.astro/.js/.jsx/.ts/.tsx/.md/.mdx/.json` files via Prettier.

## Architecture

Astro 4 static site (SSG) — every route resolves to HTML at build time.

### Content pipeline

Posts live in `src/content/blog/*.{md,mdx}` and are validated by the Zod schema in `src/content/config.ts`. Required frontmatter: `title` (max 80 chars), `description`, `pubDate`, `heroImage` (resolved as an Astro `image()` asset), `category`, `tags`. `draft: true` excludes a post from `getPosts()` and related helpers.

`category` is a Zod `enum` over `CATEGORIES` in `src/data/categories.ts`. **Adding or renaming a category requires editing both `src/data/categories.ts` and the matching Tina option list in `tina/config.ts`** — Tina reads the same constant, but the build will fail with a Zod error if a post references a category not in the array.

All content queries go through `src/utils/post.ts` (re-exported via `src/utils/index.ts`):
- `getPosts(max?)` — drafts filtered out, sorted by `pubDate` desc. Use this rather than calling `getCollection('blog')` directly so draft filtering stays consistent.
- `getCategories`, `getTags`, `getPostByTag`, `filterPostsByCategory` — same draft-filtering invariant.

Note: tags are normalized to lowercase on read (`getTags`, `getPostByTag`) but stored as-authored in frontmatter; preserve this when adding tag-related features.

### Routing

- `src/pages/index.astro` — home (latest N posts).
- `src/pages/post/[...slug].astro` — individual post; uses `getStaticPaths` over `getPosts()`. Related posts are computed by tag overlap (lowercased), capped at 3.
- `src/pages/category/[category]/[page].astro` — paginated category index, page size from `siteConfig.paginationSize`.
- `src/pages/tags/index.astro` and `src/pages/tags/[...tag]/index.astro` — tag listings.
- `src/pages/rss.xml.ts` — RSS feed (via `@astrojs/rss`).

### Configuration surface

- `src/data/site.config.ts` — site URL, author, title, description, `paginationSize`. `astro.config.mjs` reads `siteConfig.site` for the canonical `site` field, so update here rather than in the Astro config.
- `src/data/links.ts` — social/nav links.
- `src/data/disqus.config.ts` — Disqus comments toggle/config; the post layout only renders Disqus when `disqusConfig.enabled` is true.
- `astro.config.mjs` — integrations: `mdx`, `sitemap`, `tailwind`. `drafts: true` is enabled in both markdown and MDX config so drafts render in dev (the runtime filter in `getPosts` is what actually hides them from listings).
- `src/utils/readTime.ts` — `remarkReadingTime` plugin; exposes `minutesRead` via `remarkPluginFrontmatter` (consumed in `src/layouts/BlogPost.astro`).

### TypeScript path aliases (see `tsconfig.json`)

- `@/components/*` → `src/components/*.astro` (note: the `.astro` extension is **baked into the alias**, so importing non-`.astro` files from `components/` won't resolve through this alias)
- `@/layouts/*` → `src/layouts/*.astro` (same `.astro`-baked-in pattern)
- `@/utils` → `src/utils/index.ts` (barrel only — no `@/utils/*`)
- `@/data/*` → `src/data/*`
- `@/site-config` → `src/data/site.config.ts`
- `@/styles` → `src/styles/`

### Styling & client behavior

Tailwind with `@tailwindcss/typography` (the `prose` classes in `BlogPost.astro` drive markdown rendering). Theme toggle lives in `ProviderTheme.astro` (persists via `localStorage`). `ProviderAnimations.astro` + the `motion` library gate animations behind `localStorage.getItem('animations') === 'true'`. Static client-side search is provided by Pagefind, which depends on the `postbuild` step.

### Tina CMS

`tina/config.ts` defines a single `post` collection pointing at `src/content/blog` (MDX). `clientId`/`token` are `null` — Tina runs in local-only mode. To use Tina Cloud, populate these (typically from `tina.io`). Tina exposes a custom `SButton` rich-text template; the matching MDX component is registered in `src/pages/post/[...slug].astro` via `<Content components={{ pre: Code, SButton }} />`.
