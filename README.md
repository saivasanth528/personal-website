# saivasanth.com — Personal Portfolio

Live at [saivasanth.com](https://saivasanth.com)

---

## What this is

A personal portfolio site built to showcase my work as a Staff Software Engineer. Designed with a dark, minimal aesthetic — fast to load, easy to read, and built to last without constant maintenance.

---

## Framework: Astro

This site is built with **[Astro v4](https://astro.build)**.

### Why Astro over React/Next.js/plain HTML?

A portfolio is primarily a static content site — there's no user login, no real-time data, no complex state. Most JavaScript frameworks (React, Vue) ship a JS runtime to the browser even when you don't need one. That's unnecessary overhead for a site that's mostly text and layout.

Astro takes the opposite approach: **zero JavaScript by default**. It renders everything to plain HTML at build time. JS only ships to the browser when you explicitly need interactivity (like the sticky scroll experience section).

| Concern | Plain HTML | React/Next.js | Astro |
|---|---|---|---|
| Build complexity | None | High | Low |
| JS bundle to browser | None | Always | Only what you opt into |
| Component reuse | Manual copy-paste | Yes | Yes |
| Static site generation | Manual | Supported | First-class |
| Blog / content support | Manual | Via libraries | Built-in |

The key benefit here: **component-based authoring** (each section is its own `.astro` file) without paying the React runtime tax. The entire site ships as static HTML + CSS, with small vanilla JS scripts only where needed.

### Astro component structure

Each section of the page is a self-contained `.astro` file under `src/components/`:

```
src/
  components/
    Nav.astro        # Fixed navigation bar
    Hero.astro       # Landing section with name, tagline, social links
    About.astro      # Bio, stats, fun fact, resume download
    Experience.astro # Sticky scroll experience section
    Skills.astro     # Tech stack grouped by category
    Projects.astro   # Selected work
    Contact.astro    # Social links and contact
  layouts/
    Layout.astro     # Base HTML shell, fonts, global CSS tokens
  pages/
    index.astro      # Assembles all components into the home page
```

An `.astro` file has three parts:

```astro
---
// 1. Frontmatter: JavaScript/TypeScript that runs at BUILD TIME only.
//    Use this for data, imports, logic. Never reaches the browser.
const message = "Hello";
---

<!-- 2. HTML template: like JSX but outputs plain HTML -->
<p>{message}</p>

<style>
  /* 3. Scoped CSS: automatically scoped to this component only.
        No class name collisions across components. */
  p { color: red; }
</style>
```

---

## Styling: Tailwind CSS + CSS custom properties

Tailwind is included but used sparingly — mostly for utility resets. The bulk of styling is hand-written CSS inside each component's `<style>` block, using **CSS custom properties (variables)** defined in `Layout.astro` as the design system:

```css
:root {
  --bg: #09090b;
  --text: #f4f4f5;
  --muted: #a1a1aa;
  --accent: #06b6d4;    /* cyan */
  --accent-2: #8b5cf6;  /* purple */
  --border: #27272a;
}
```

Changing the accent colour site-wide is one variable change.

---

## Interesting implementation details

### Sticky scroll experience section

The experience section (`Experience.astro`) creates a scroll-driven panel switcher without any library:

```
┌─────────────────────────────────┐
│  outer wrapper: height = 3×100vh│  ← creates scroll room
│  ┌───────────────────────────┐  │
│  │ inner: position: sticky   │  │  ← stays in viewport
│  │ top: 0; height: 100vh     │  │
│  │                           │  │
│  │  [nav]    [content panel] │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

The scroll math:
```js
const progress = scrolled / scrollable;          // 0 → 1 as you scroll
const index = Math.floor(progress * count);      // which panel to show
progressFill.style.height = `${progress * 100}%`; // animate the side bar
```

No library. No IntersectionObserver. Just `window.scrollY` and a sticky container.

### set:html for rich highlight content

Experience highlights that contain inline links use Astro's `set:html` directive instead of `{h}`:

```astro
<span set:html={h}></span>
```

This is needed because `{h}` escapes HTML — so `<a href="...">` would render as literal text. `set:html` tells Astro to trust the HTML string. Inline links (release notes, press coverage) are embedded directly in the data strings as template literals.

**Gotcha:** Astro scopes component CSS using a data attribute (e.g. `[data-astro-cid-xxx]`). HTML injected via `set:html` doesn't get that attribute, so scoped styles won't match it. Fix: wrap those rules in `:global()`.

---

## Deployment: GitHub Actions → GitHub Pages

Every push to `master` triggers an automated deploy. Here's the full pipeline:

```
git push → GitHub Actions triggered
              │
              ▼
         ┌─────────┐
         │  build  │  ubuntu-latest
         │─────────│  1. actions/checkout@v4  (clone repo)
         │         │  2. actions/setup-node@v4 (Node 20, npm cache)
         │         │  3. npm ci               (clean install)
         │         │  4. npm run build        (astro build → dist/)
         │         │  5. upload-pages-artifact (zip dist/ for deploy job)
         └────┬────┘
              │ artifact passed via GitHub's internal storage
              ▼
         ┌──────────┐
         │  deploy  │  needs: build
         │──────────│  actions/deploy-pages@v4
         │          │  → pushes artifact to GitHub Pages CDN
         └──────────┘
              │
              ▼
     https://saivasanth.com  (live in ~30 seconds)
```

### Why two separate jobs (build + deploy)?

GitHub Pages deployment requires specific **permissions** (`pages: write`, `id-token: write`). Separating build from deploy means the dangerous permissions are scoped only to the deploy job. The build job runs with minimal read-only permissions — safer by design.

### workflow_dispatch

The workflow also has `workflow_dispatch: ` in the trigger. This adds a "Run workflow" button in the GitHub Actions UI so you can manually trigger a deploy without pushing a commit. Useful when you update a Google Drive resume link (no code change needed, but you want to redeploy).

### Custom domain + HTTPS

The custom domain is configured via `public/CNAME`:
```
saivasanth.com
```

Astro copies everything in `public/` directly to `dist/` at build time, so the CNAME file gets included in every deploy. GitHub Pages reads it, configures the domain, and provisions a Let's Encrypt TLS certificate automatically.

DNS is set up with four A records pointing to GitHub's Pages IPs and a CNAME for `www`. GitHub handles the HTTPS redirect from `www` to the apex domain.

---

## Local development

```bash
npm install
npm run dev       # starts dev server at localhost:4321
npm run build     # production build to dist/
npm run preview   # preview the production build locally
```

---

## Resume hosting

The resume is hosted on Google Drive (not in this repo). The link in `About.astro` points to a shareable Drive URL. When the resume needs updating, use **Manage versions** in Drive to replace the file — the URL stays the same, no redeploy needed.
