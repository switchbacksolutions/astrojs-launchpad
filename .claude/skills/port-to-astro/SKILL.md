---
name: port-to-astro
description: Port an existing website to the Astro template in the current working directory. Use when the user provides a URL and wants it migrated to Astro. Invoked via `/port-to-astro <url>`.
---

Port an existing website to the current Astro template. The argument is a URL (e.g. `/port-to-astro https://example.com`).

## Step 1 — Fetch the source site

Try these in order, stopping at the first success:

1. **WebFetch** — attempt `WebFetch(url)` on the provided URL.
2. **curl via Bash** — if WebFetch fails or returns an error/block page, run:
   ```
   curl -sL --max-time 15 -A "Mozilla/5.0" "<URL>"
   ```
3. **Wayback Machine** — if curl also fails, try WebFetch on `https://web.archive.org/web/2/<URL>` to retrieve a cached snapshot.
4. **Manual fallback** — if all three fail, stop and tell the user:
   > "I couldn't retrieve that site automatically. Please run `! curl -sL <URL>` in the prompt — the output will land here and I can continue from there."

Once you have HTML, proceed.

## Step 2 — Audit the source site

First, find all `<link rel="stylesheet">` tags in the HTML and fetch each stylesheet using the same fallback chain as Step 1 (WebFetch → curl → Wayback Machine). Collect all stylesheet content before proceeding.

Then analyze the HTML and stylesheets and answer these questions (show a brief summary to the user):

- **Site type**: Is this a blog, marketing site, portfolio, or something else?
- **Pages detected**: List the pages/sections visible in the nav or linked from the home page.
- **Blog/content**: Are there blog posts or news articles? If so, how many are visible and what are their titles/dates?
- **Nav links**: What are the top-level navigation items?
- **Footer links**: What links appear in the footer?
- **Key copy**: What is the hero headline, sub-headline, and CTA?
- **Images**: Note any hero images or logos referenced.
- **Stylesheets fetched**: List each stylesheet URL and whether it was successfully retrieved.

Ask the user to confirm or correct the audit before writing any files.

## Step 3 — Map to the Astro template

Using the audit, plan the following (confirm with user before executing):

| Source element | Astro target |
|---|---|
| Home page content | `src/pages/index.astro` |
| About/team page | `src/pages/about.astro` |
| Blog posts | `src/content/blog/<slug>.md` each |
| Other static pages | `src/pages/<slug>.astro` each |
| Nav links | `NAV_LINKS` in `src/components/Header.astro` |
| Footer links | `FOOTER_LINKS` in `src/components/Footer.astro` |
| Site title/description | `astro.config.mjs` `site` + `BaseHead` defaults |

If blog posts exist but their full content isn't on the home page, ask the user whether to:
- (a) Fetch each post URL individually and port the full content, or
- (b) Create stub posts with just the title/date/description and a `draft: true` flag.

Also ask:
> "Do you have a Google Analytics Measurement ID for this site? (format: G-XXXXXXXXXX — leave blank to skip)"

Record the answer for use in Step 5.

## Step 4 — Style plan

Using the fetched stylesheets and HTML, extract the site's design tokens and produce a style plan. Present it to the user and ask for confirmation before proceeding.

### Colors
Find the primary, secondary, accent, background, and text colors (from CSS custom properties, hardcoded hex/rgb values, or framework config). Map each to the closest Tailwind color class (e.g. `bg-blue-600`) or, if there's no close match, propose a CSS custom property to add to `global.css` under `@layer base`.

### Typography
Identify font families and the heading/body size scale. Map to Tailwind's `font-sans`/`font-serif`/`font-mono` or note a Google Fonts import that should be added to `BaseHead.astro`. Map font sizes to Tailwind's type scale (`text-sm`, `text-xl`, etc.).

### Spacing & layout
Note any non-standard max-widths, padding scales, or gap values that differ from Tailwind defaults. Propose `@layer components` additions to `global.css` for any repeated patterns (e.g. `.section-container`, `.card`).

### Dark mode
If the original site has a dark mode, note which color swaps are needed and plan `dark:` variants for each.

### Style plan output format
Present the plan as a short table or bullet list per category. Example:

```
Colors
  Primary:    #1a56db  →  bg-blue-600 / text-blue-600
  Background: #f9fafb  →  bg-gray-50
  ...

Typography
  Body font:  Inter  →  add Google Fonts <link> to BaseHead.astro; use font-sans
  ...

global.css additions
  .section-container { @apply mx-auto max-w-5xl px-4 sm:px-6; }
  ...
```

Wait for user approval before moving to Step 5.

## Step 5 — Execute the port

If the user provided a Google Analytics Measurement ID in Step 3, enable it by setting the environment variable. The template's `BaseHead.astro` already reads `PUBLIC_GA_MEASUREMENT_ID` at build time and injects the GA snippet automatically when the variable is set.

1. Add or uncomment the line in `.env` (create the file from `.env.example` if it doesn't exist):
   ```
   PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX
   ```
2. Uncomment the same line in `.env.example` and fill in the ID so it's documented for future developers.
3. Remind the user to add `PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX` as an environment variable in **Netlify → Site settings → Environment variables** so it takes effect in production builds.

The CSP in `netlify.toml` already permits GA domains — no additional changes needed.



Read each target file before editing it. Follow the conventions in CLAUDE.md exactly:

- Use `@/` path aliases for all imports.
- Use Tailwind utility classes for all styling — no inline `style` attributes.
- Use `dark:` variants wherever the original site has a dark-mode equivalent.
- Blog post frontmatter must satisfy the Zod schema in `src/content/config.ts` (title, description, pubDate required; draft defaults false).
- Do NOT remove existing placeholder content from the template unless replacing it with real content.
- Do NOT add `console.log`, comments, or docstrings to unchanged code.

For each page, preserve the original site's text and structure as closely as the Astro template allows. Adapt layout to the template's conventions rather than trying to pixel-match the original design.

## Step 6 — Verify

After all files are written, run:
```
npm run check
```
Report any type errors and fix them before finishing. Do NOT run `npm run build` unless the user asks.

Tell the user what was ported and list any content that was skipped or needs manual attention (e.g. contact forms, embedded third-party widgets, gated content).
