# Static Home Page for Lovable Sites

## A phased guide to making your React/Vite home page visible to search engines and AI crawlers

**Scope:** Home page only (`/` route). We are deliberately not solving SSG for all routes — that's a different, larger problem.

**The problem:** Lovable sites ship as React SPAs. When a search engine or AI bot requests your home page, it receives an empty `<div id="root"></div>` and no content. Your home page is invisible to Google, Bing, ChatGPT, Claude, and every other crawler that doesn't execute JavaScript.

**Who this is for:** Anyone with a Lovable-hosted React/Vite site using React Router who wants their home page indexed.

---

## How to use this doc

This guide is structured as a **technical spike** followed by **three escalating phases**. Start with the spike. If it works, you're done. If it doesn't, move to Phase 1. Each phase is a self-contained fallback — you stop as soon as one works reliably for your site.

```
Spike: Inject static HTML at build time (minutes, zero infra)
  ↓ didn't work?
Phase 1: Prerender with Puppeteer postbuild script (hours, CLI required)
  ↓ didn't work?
Phase 2: Cloudflare Worker as a prerender proxy (half-day, free tier)
  ↓ didn't work?
Phase 3: Dedicated static home page outside React (1-2 days, fully decoupled)
```

---

## Spike: Static HTML Injection (The 15-Minute Test)

**Goal:** Put real HTML content inside `index.html` so crawlers see your home page content even before React mounts.

**Concept:** Your `index.html` already has `<div id="root"></div>`. React will overwrite whatever is inside that div when it hydrates. So we can **pre-fill it with a static snapshot of your home page content** — crawlers see real HTML, users still get the full React app.

### Steps (Lovable editor — no CLI needed)

1. Open your live home page in Chrome
2. Right-click → Inspect → select the contents inside `<div id="root">`
3. Copy the rendered HTML (right-click the element → Copy → Copy outerHTML)
4. In your Lovable project, open `index.html`
5. Paste that HTML inside `<div id="root">...</div>`
6. Also add the critical SEO tags to `<head>` if missing:

```html
<head>
  <!-- REQUIRED: Update these for your site -->
  <title>Your Site Name — Your Value Proposition</title>
  <meta name="description" content="150 chars max describing your home page." />
  <link rel="canonical" href="https://yoursite.com/" />

  <!-- Open Graph -->
  <meta property="og:title" content="Your Site Name" />
  <meta property="og:description" content="Same or similar to meta description." />
  <meta property="og:image" content="https://yoursite.com/og-image.jpg" />
  <meta property="og:url" content="https://yoursite.com/" />
  <meta property="og:type" content="website" />

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" />
</head>
```

7. Deploy and test:

```bash
# From terminal:
curl -s https://yoursite.com | head -100

# Or use the SEO Health Scanner to verify
```

### Does it work?

**Yes if:** `curl` returns your actual home page content (headings, text, links). React hydrates cleanly on top without visual flicker or console errors.

**No if:** React hydration throws mismatches, your home page content changes frequently (making the static snapshot stale fast), or Lovable's build process overwrites your `index.html` edits.

### Limitations

- **Manual process.** Every time your home page content changes, you need to re-copy the HTML. Fine for landing pages that change monthly, not for dynamic content.
- **Hydration mismatches.** React may warn in the console if the static HTML doesn't exactly match what React renders. Usually harmless for SEO (crawlers don't run React), but test in browser.
- **Lovable rebuilds.** If Lovable regenerates `index.html` on deploy, your edits get overwritten. If this happens, skip to Phase 1.

---

## Phase 1: Prerender at Build Time (Puppeteer Postbuild)

**Goal:** Automatically generate a static HTML snapshot of your home page after every build, so it's always in sync with your React app.

**Requires:** CLI access (clone repo from Lovable → run locally or in CI).

### How it works

After `vite build` produces your SPA in `dist/`, a postbuild script launches a headless browser, loads `dist/index.html`, waits for React to render, and saves the fully-rendered HTML back to `dist/index.html`.

### Setup

#### 1. Install Puppeteer

```bash
npm install --save-dev puppeteer
```

#### 2. Create `scripts/prerender-home.mjs`

```javascript
import puppeteer from 'puppeteer';
import { readFile, writeFile } from 'fs/promises';
import { resolve } from 'path';
import { createServer } from 'http';

const DIST = resolve('dist');
const PORT = 4173;

// Serve the dist folder locally
async function serve() {
  const handler = async (req, res) => {
    const url = req.url === '/' ? '/index.html' : req.url;
    try {
      const file = await readFile(resolve(DIST, `.${url}`));
      const ext = url.split('.').pop();
      const types = { html: 'text/html', js: 'application/javascript', css: 'text/css', json: 'application/json' };
      res.writeHead(200, { 'Content-Type': types[ext] || 'application/octet-stream' });
      res.end(file);
    } catch {
      // SPA fallback — serve index.html for all routes
      const index = await readFile(resolve(DIST, 'index.html'));
      res.writeHead(200, { 'Content-Type': 'text/html' });
      res.end(index);
    }
  };
  const server = createServer(handler);
  return new Promise(r => server.listen(PORT, () => r(server)));
}

async function prerender() {
  console.log('🔍 Prerendering home page...');
  const server = await serve();

  const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
  const page = await browser.newPage();

  await page.goto(`http://localhost:${PORT}/`, { waitUntil: 'networkidle0', timeout: 30000 });

  // Wait for React to finish rendering — adjust selector to match your home page
  await page.waitForSelector('h1, [data-testid="home"], main', { timeout: 10000 }).catch(() => {
    console.warn('⚠️  Could not find expected home page element. Proceeding anyway.');
  });

  // Extract the full HTML
  const html = await page.content();

  // Inject a marker so we can verify prerendering worked
  const marked = html.replace('</head>', '<meta name="prerender-status" content="prerendered" />\n</head>');

  await writeFile(resolve(DIST, 'index.html'), marked, 'utf-8');
  console.log('✅ dist/index.html updated with prerendered home page content.');

  await browser.close();
  server.close();
}

prerender().catch(err => {
  console.error('❌ Prerender failed:', err.message);
  process.exit(1);
});
```

#### 3. Add to `package.json`

```json
{
  "scripts": {
    "build": "vite build",
    "postbuild": "node scripts/prerender-home.mjs"
  }
}
```

Now `npm run build` automatically prerenders your home page.

#### 4. Verify

```bash
npm run build
grep -c "prerender-status" dist/index.html  # should print 1
cat dist/index.html | head -80              # should contain real content
```

### Lovable editor path (no CLI)

If you can't run scripts locally, you can still use this approach by asking Lovable's AI to add the `prerender-home.mjs` script and the `postbuild` hook. However, Puppeteer may not be available in Lovable's build environment. If the build fails, skip to Phase 2.

### Does it work?

**Yes if:** `dist/index.html` contains your rendered home page content after build, and the deployed site shows content to `curl`.

**No if:** Lovable's hosting environment doesn't support Puppeteer in the build step, or the home page relies on API calls that aren't available at build time (e.g., fetching product data from a backend).

---

## Phase 2: Cloudflare Worker Prerender Proxy

**Goal:** Intercept crawler requests at the edge and serve a cached prerendered version of your home page, without changing your Lovable build at all.

**Requires:** Free Cloudflare account. Your domain must use Cloudflare DNS (or you can use a `yoursite.workers.dev` subdomain for testing).

### How it works

A Cloudflare Worker sits in front of your Lovable site. When it detects a bot user-agent requesting `/`, it serves a prerendered HTML snapshot. Human visitors get the normal SPA.

```
Bot → Cloudflare Worker → serves cached static HTML
Human → Cloudflare Worker → passes through to Lovable SPA
```

### Setup

#### 1. Create the Worker

In Cloudflare dashboard → Workers & Pages → Create Worker:

```javascript
// List of known bot user-agents
const BOT_AGENTS = [
  'googlebot', 'bingbot', 'slurp', 'duckduckbot', 'baiduspider',
  'yandexbot', 'facebot', 'twitterbot', 'linkedinbot', 'whatsapp',
  'telegrambot', 'applebot', 'gptbot', 'claudebot', 'anthropic-ai',
  'bytespider', 'perplexitybot', 'cohere-ai',
];

function isBot(userAgent) {
  const ua = (userAgent || '').toLowerCase();
  return BOT_AGENTS.some(bot => ua.includes(bot));
}

// Your prerendered home page HTML — paste the full static version here
const PRERENDERED_HOME = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Your Site — Your Value Prop</title>
  <meta name="description" content="Your 150-char description." />
  <link rel="canonical" href="https://yoursite.com/" />
  <meta property="og:title" content="Your Site" />
  <meta property="og:description" content="Your description." />
  <meta property="og:image" content="https://yoursite.com/og-image.jpg" />
  <meta property="og:url" content="https://yoursite.com/" />
  <meta property="og:type" content="website" />
  <meta name="twitter:card" content="summary_large_image" />
</head>
<body>
  <!-- PASTE YOUR STATIC HOME PAGE HTML HERE -->
  <main>
    <h1>Your Headline</h1>
    <p>Your content that crawlers need to see.</p>
  </main>
</body>
</html>`;

export default {
  async fetch(request) {
    const url = new URL(request.url);
    const ua = request.headers.get('user-agent') || '';

    // Only intercept the home page for bots
    if (url.pathname === '/' && isBot(ua)) {
      return new Response(PRERENDERED_HOME, {
        headers: {
          'Content-Type': 'text/html; charset=utf-8',
          'X-Prerender': 'static-home',
          'Cache-Control': 'public, max-age=3600',
        },
      });
    }

    // Everything else: pass through to your Lovable site
    return fetch(request);
  },
};
```

#### 2. Route the Worker

In Cloudflare DNS, point your domain to Lovable's hosting. Add a Worker Route for `yoursite.com/*` that triggers this Worker.

#### 3. Verify

```bash
# Simulate Googlebot
curl -s -A "Googlebot" https://yoursite.com/ | head -50

# Compare to normal browser request
curl -s https://yoursite.com/ | head -50
```

### Automating the snapshot refresh

The static HTML in the Worker is manual — same as the Spike. To automate it:

- Set up a weekly cron (GitHub Action, or Cloudflare Cron Trigger) that uses Puppeteer to render your live home page and updates the Worker's `PRERENDERED_HOME` variable via the Cloudflare API.
- Or use Cloudflare KV to store the snapshot, and update it via an API endpoint.

### Does it work?

**Yes if:** Bot requests to `/` return full HTML content. Human visitors get the normal React SPA. You can update the snapshot on a schedule that matches your content change frequency.

**No if:** You can't move DNS to Cloudflare, or maintaining a separate static snapshot is too much operational overhead.

---

## Phase 3: Decoupled Static Home Page

**Goal:** Build your home page as a plain HTML file completely outside of React. Your SPA handles everything else.

**This is the nuclear option** — maximum reliability, maximum effort. Use this when the React SPA fundamentally resists prerendering (complex auth flows, heavy client-side data fetching, animation libraries that break in headless browsers, etc.).

### How it works

```
/ (home page)     → static index.html (hand-crafted or templated)
/app/*            → React SPA (unchanged)
/about, /pricing  → React SPA (unchanged)
```

The home page is a standalone HTML file with zero React. It links to your SPA routes normally. Users clicking "Get Started" or nav links enter the React app.

### Setup

#### 1. Create `public/index.html`

In Vite, files in `public/` are served as-is. But there's a conflict: Vite also uses `index.html` at the root as its entry point.

**Solution:** Restructure so the SPA entry point is on a different path:

```
public/
  index.html          ← your static home page (served at /)
  robots.txt
  sitemap.xml
src/
  main.tsx            ← React app entry
index.html            ← Vite entry point (rename to app.html via config)
```

#### 2. Configure Vite

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      input: {
        app: 'index.html',  // React SPA entry
      },
    },
  },
});
```

#### 3. Build the static home page

Your `public/index.html` is a complete, self-contained HTML page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Your Site — Your Value Proposition</title>
  <meta name="description" content="150 chars max." />
  <link rel="canonical" href="https://yoursite.com/" />

  <meta property="og:title" content="Your Site" />
  <meta property="og:description" content="Your description." />
  <meta property="og:image" content="https://yoursite.com/og-image.jpg" />
  <meta property="og:url" content="https://yoursite.com/" />
  <meta property="og:type" content="website" />
  <meta name="twitter:card" content="summary_large_image" />

  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Your Site",
    "url": "https://yoursite.com",
    "description": "What your site does."
  }
  </script>

  <style>
    /* Inline your critical CSS here — keep it self-contained */
  </style>
</head>
<body>
  <header>
    <nav>
      <a href="/">Home</a>
      <a href="/about">About</a>
      <a href="/pricing">Pricing</a>
      <a href="/app">Get Started</a>
    </nav>
  </header>

  <main>
    <h1>Your Main Headline</h1>
    <p>Your value proposition. This is what Google and AI crawlers will index.</p>
    <a href="/app">Get Started →</a>
  </main>

  <footer>
    <p>© 2026 Your Company</p>
  </footer>
</body>
</html>
```

#### 4. Handle routing on the host

On Lovable's built-in hosting, you may need a `_redirects` or routing config to ensure:
- `/` serves `public/index.html` (static)
- All other routes serve the SPA's `index.html` (React Router handles them)

If Lovable uses Netlify-style redirects:
```
# _redirects
/app/*    /index.html    200
/about    /index.html    200
/pricing  /index.html    200
```

### Lovable editor path

This is fully doable in the Lovable editor — you're just creating an HTML file in `public/` and editing `vite.config.ts`. No CLI tools required.

### Tradeoffs

**Pros:** Bulletproof SEO. Zero hydration issues. Fastest possible load for the home page. Full control over every tag. Works with any hosting.

**Cons:** Two sources of truth for home page design. Visual changes require updating both the static HTML and (if the SPA also renders a home view) the React component. You're maintaining two codebases for one page.

**Mitigation:** If your home page is mostly a marketing/landing page, it changes infrequently and the maintenance cost is low. If it's highly dynamic, this approach is wrong — go back to Phase 2.

---

## Quick Reference: What to verify after any phase

After implementing any phase, run these checks to confirm it worked:

```bash
# 1. Does curl see real content?
curl -s https://yoursite.com/ | grep -i "<h1>"

# 2. Is the title tag correct?
curl -s https://yoursite.com/ | grep -i "<title>"

# 3. Are meta tags present?
curl -s https://yoursite.com/ | grep -i "og:title"
curl -s https://yoursite.com/ | grep -i "description"

# 4. Google's test
#    → https://search.google.com/test/rich-results
#    Enter your URL and check the rendered HTML tab

# 5. Run the SEO Health Scanner
#    Open seo-health-scanner.html and scan your URL
```

## Decision flowchart

```
Does your home page content change more than once a month?
├── No → Spike (manual static injection) — try this first
│     └── Lovable overwrites your edits?
│           └── Phase 1 (Puppeteer postbuild)
└── Yes → Phase 1 (automated prerender)
      └── Puppeteer won't run in your build env?
            └── Phase 2 (Cloudflare Worker proxy)
                  └── Can't use Cloudflare / too much maintenance?
                        └── Phase 3 (decoupled static page)
```

---

## Files checklist (all phases)

Regardless of which phase you use, make sure these files exist at your domain root:

| File | Purpose | How to verify |
|------|---------|---------------|
| `/robots.txt` | Tells crawlers what to index | `curl https://yoursite.com/robots.txt` |
| `/sitemap.xml` | Lists all indexable URLs | `curl https://yoursite.com/sitemap.xml` |
| `/` (index.html) | Contains real HTML content | `curl https://yoursite.com/ \| grep "<h1>"` |

### Minimal `robots.txt`

Place in `public/robots.txt`:

```
User-agent: *
Allow: /

Sitemap: https://yoursite.com/sitemap.xml
```

### Minimal `sitemap.xml`

Place in `public/sitemap.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://yoursite.com/</loc>
    <lastmod>2026-02-14</lastmod>
    <priority>1.0</priority>
  </url>
</urlset>
```

---

*Last updated: February 2026*