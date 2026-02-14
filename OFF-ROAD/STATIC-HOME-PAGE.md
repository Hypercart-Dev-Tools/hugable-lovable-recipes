# Static Home Page & SEO Capture for Lovable Sites
**Last updated:** February 2026
**Status:** Guide + Experimental Alpha tooling

---

## TLDR: Can you replace a Lovable home page with static HTML?

Not seamlessly — Lovable's SPA architecture has no native path for this. Read the full guide below to understand the constraints and pick your approach:

- **Cloudflare Worker** (free, DIY) — documented in this guide, Part 2 Option C
- **[lovablehtml.com](https://lovablehtml.com)** — third-party service built specifically for this problem

---

## Table of Contents

- [TLDR](#tldr-can-you-replace-a-lovable-home-page-with-static-html)
- [LLM Instructions](#llm-instructions)
- [The Problem](#make-your-reactvite-site-visible-to-search-engines-and-ai-crawlers)
- [How to Use This Doc](#how-to-use-this-doc)
- **Part 1: Non-Technical Path**
  - [Step 1: Create Your SEO Test Page](#step-1-create-your-seo-test-page)
  - [Step 2: Verify It Works](#step-2-verify-it-works)
  - [Step 3: Apply to Your Home Page](#step-3-apply-to-your-home-page)
- **Part 2: Technical Path**
  - [Option A: SEO Capture Tool](#option-a-seo-capture-tool-experimental-alpha)
  - [Option B: Prerender at Build Time](#option-b-prerender-at-build-time-puppeteer-postbuild)
  - [Option C: Cloudflare Worker (Recommended)](#option-c-cloudflare-worker-prerender-proxy-recommended-for-lovable)
- **Appendix** (alphabetical)
  - [Decision Flowchart](#decision-flowchart)
  - [Embedded Scripts (Source of Truth)](#embedded-scripts-source-of-truth)
  - [Files Checklist](#files-checklist)
  - [Lessons Learned](#lessons-learned-on-this-project)

---

## LLM Instructions

> **For AI assistants (AGENTS.md context):** This document is a self-contained guide and toolkit for converting a React/Vite SPA home page into a static HTML page that search engines and AI crawlers can read. It contains two embedded scripts (Node.js + Bash) that should be extracted into the user's project when requested.
>
> **Available tasks:**
>
> 1. **Extract scripts** — When the user wants to use the SEO Capture Tool, extract the two scripts from the "Embedded Scripts" section into `scripts/capture-spa.mjs` and `scripts/capture-and-verify.sh`. Add the npm scripts to `package.json`. Install Puppeteer as a devDependency. Make the bash script executable.
>
> 2. **Run capture** — After scripts are extracted, run `node scripts/capture-spa.mjs <url>` to capture a live SPA page as self-contained static HTML.
>
> 3. **Full SEO walkthrough** — Walk the user through the entire flow: create `/seo-test/`, verify with crawlers, then choose a promotion strategy (manual copy or Cloudflare Worker).
>
> **When extracting scripts:** Copy the code blocks exactly as written. Do not modify the logic. Create the `scripts/` directory if it doesn't exist. The `package.json` additions are:
> ```json
> {
>   "scripts": {
>     "capture": "node scripts/capture-spa.mjs",
>     "capture:verify": "./scripts/capture-and-verify.sh"
>   },
>   "devDependencies": {
>     "puppeteer": "^24.37.3"
>   }
> }
> ```

### For users: getting the scripts into your project

Tell your AI assistant:

> "Please read the STATIC-HOME-PAGE.md doc and extract the SEO Capture Tool scripts into my project. Create `scripts/capture-spa.mjs` and `scripts/capture-and-verify.sh` from the embedded code blocks, add the npm scripts to my package.json, and install Puppeteer."

Your AI will pull the code directly from this document — no copy-pasting needed.

---

## Make your React/Vite site visible to search engines and AI crawlers

**The problem:** Lovable sites ship as React SPAs. When a search engine or AI bot requests your site, it receives an empty `<div id="root"></div>` and no content. Your site is invisible to Google, Bing, ChatGPT, Claude, and every other crawler that doesn't execute JavaScript.

**The approach:** Instead of modifying your live home page (risky — Lovable can overwrite it, React hydration can break), we create a **standalone test page at `/seo-test/`** first. This page lives safely in `public/seo-test/index.html`, completely outside React. Verify it works, then decide whether to promote it to `/` or use it as a template for a Cloudflare Worker.

**Who this is for:** Anyone with a Lovable-hosted React/Vite site who wants their content indexed.

---

## How to use this doc

Pick your path based on your comfort level:

```
Non-Technical (Lovable editor only, no CLI)
  Step 1: Create a static test page at /seo-test/
  Step 2: Verify crawlers can see it
  Step 3: Apply the pattern to your home page

Technical (CLI, CI/CD, or Cloudflare)
  Option A: SEO Capture Tool — automated capture from a live/dev site
  Option B: Prerender at build time — Puppeteer postbuild script
  Option C: Cloudflare Worker — proxy for bots at the edge
```

**Everyone starts with the non-technical path.** The technical options are for automating or scaling what you proved works in the test.

---

# Part 1: Non-Technical Path

**Tools needed:** Lovable editor + a browser. No CLI, no terminal, no installs.

## Step 1: Create your SEO test page

### Why `/seo-test/` instead of modifying `/`?

- **Zero risk.** Your live home page stays untouched. Nothing can break.
- **No hydration conflicts.** This page is plain HTML — React never touches it.
- **Lovable-proof.** Files in `public/` survive Lovable rebuilds. Your root `index.html` does not.
- **Verifiable.** You can test with real crawlers before committing to anything.

### Steps (Lovable editor)

1. **Create the folder and file.** In your Lovable project, create:
   ```
   public/seo-test/index.html
   ```

2. **Paste this template** and customize it for your site:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- REQUIRED: Update these for your site -->
  <title>Your Site Name — Your Value Proposition</title>
  <meta name="description" content="150 chars max describing what your site does." />
  <!-- Canonical points to / (your real home page), not /seo-test/.
       This tells Google that / is the authoritative URL, even though
       the content lives here during testing. -->
  <link rel="canonical" href="https://yoursite.com/" />

  <!-- Open Graph (shows when shared on social media) -->
  <meta property="og:title" content="Your Site Name" />
  <meta property="og:description" content="Same or similar to meta description." />
  <meta property="og:image" content="https://yoursite.com/og-image.jpg" />
  <meta property="og:url" content="https://yoursite.com/" />
  <meta property="og:type" content="website" />

  <!-- Twitter/X Card -->
  <meta name="twitter:card" content="summary_large_image" />

  <!-- Structured data (helps Google understand your site).
       Change @type to match your site: "Organization" for a company,
       "SoftwareApplication" for a SaaS product, "Product" for e-commerce,
       "LocalBusiness" for a local service, etc. -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Your Site Name",
    "url": "https://yoursite.com",
    "description": "What your site does."
  }
  </script>

  <style>
    /* Inline your styles here — this page is self-contained */
    body { font-family: system-ui, sans-serif; margin: 0; padding: 0; }
    header { padding: 1rem 2rem; border-bottom: 1px solid #eee; }
    nav a { margin-right: 1rem; text-decoration: none; color: #333; }
    main { max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
    footer { text-align: center; padding: 2rem; color: #666; font-size: 0.875rem; }
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
    <p>Your value proposition. This is what Google and AI crawlers will see and index.</p>
    <p>Add your real content here — describe what your site does, who it's for,
       and why someone should use it. This is your SEO content.</p>
    <a href="/app">Get Started</a>
  </main>

  <footer>
    <p>&copy; 2026 Your Company</p>
  </footer>
</body>
</html>
```

3. **Customize the content.** Replace all placeholder text with your actual:
   - Site name and headline
   - Value proposition / description
   - Navigation links
   - Any key content you want crawlers to index

4. **Add at least one image.** The template above is text-only. A hero image significantly improves both SEO and social sharing. Add it inside `<main>`:
   ```html
   <img src="/hero.webp" alt="Short description of what the image shows"
        width="800" height="450" loading="lazy" />
   ```
   - Use **WebP format** for smaller file sizes (`.webp` instead of `.jpg`/`.png`)
   - Always include a descriptive **`alt` attribute** — this is what Google Images indexes
   - Set **`width` and `height`** to prevent layout shift (CLS) — use the image's actual dimensions
   - Place the image file in your `public/` folder so it's served as a static asset

5. **Deploy.** Push/deploy through Lovable as normal.

## Step 2: Verify it works

After deploying, open these URLs in your browser:

| What to check | How | What you should see |
|---|---|---|
| Page loads | Visit `https://yoursite.com/seo-test/` | Your static page with real content |
| Crawler sees content | View page source (Ctrl+U / Cmd+U) | Full HTML with your text visible in the source |
| Meta tags present | View source, search for `og:title` | Your Open Graph and description tags |
| Structured data | [Google Rich Results Test](https://search.google.com/test/rich-results) → enter your `/seo-test/` URL | No errors, your schema detected |
| SEO health check | [Free SEO Check](https://new2seo.com/free-seo-check) → enter your `/seo-test/` URL | Title, description, and OG tags all passing |

**If you have terminal access**, these commands also work:

```bash
# Does the page have real content?
curl -s https://yoursite.com/seo-test/ | grep -i "<h1>"

# Are meta tags present?
curl -s https://yoursite.com/seo-test/ | grep -i "og:title"

# Is the title correct?
curl -s https://yoursite.com/seo-test/ | grep -i "<title>"
```

**Quickest check:** Run your `/seo-test/` URL through the [Free SEO Check](https://new2seo.com/free-seo-check) — it validates title, description, OG tags, and structured data in one pass.

### Does it work?

**Yes if:** You see your real content in View Source (not an empty div). The Rich Results Test shows your structured data. The Free SEO Check shows green across the board.

**No if:** The page 404s (check the file path — it must be `public/seo-test/index.html`), or Lovable's hosting doesn't serve static subfolders (unlikely, but skip to the Technical path if so).

## Step 3: Apply to your home page

Once `/seo-test/` proves the concept, you have two options:

### Option A: Copy the pattern to your home page (simple, manual)

1. Open your live home page in Chrome
2. Right-click → Inspect → find the content inside `<div id="root">`
3. Copy the outer HTML of the rendered content
4. Open your project's root `index.html`
5. Paste that HTML inside `<div id="root">...</div>`
6. Add the SEO meta tags from your test page to `<head>`
7. Deploy and verify with `curl -s https://yoursite.com/ | grep "<h1>"`

**Caveats:**
- **Lovable may overwrite this.** If Lovable regenerates `index.html` on deploy, your edits are lost. You'll need to re-apply after each Lovable update.
- **React hydration warnings.** React may log console warnings if the static HTML doesn't exactly match what it renders. Usually harmless for SEO — crawlers don't run React.
- **Manual updates.** Every time your home page content changes, re-copy the HTML.

This is fine for landing pages that change monthly. For anything more dynamic, use the Technical path.

### Option B: Keep `/seo-test/` as your indexed landing page

Instead of modifying `/`, update your SEO strategy to point crawlers to `/seo-test/`:

- Set the canonical URL to `/seo-test/` in your sitemap
- Use `/seo-test/` as the URL you submit to Google Search Console
- Link to `/seo-test/` from external sources

This is unconventional but eliminates all maintenance risk. The tradeoff is a non-root URL for your primary indexed page.

**Important caveat:** Google treats root URLs (`/`) as significantly higher authority for brand queries. If someone searches "YourBrand", Google strongly prefers to show `yoursite.com/` over `yoursite.com/seo-test/`. This option works well for niche or long-tail content, but **not as your primary brand landing page**. If brand search visibility matters to you, treat `/seo-test/` as a stepping stone and plan to promote the content to `/` using Option C below or the Technical path.

### Lovable hosting constraint: directory indexes don't resolve

> **Discovery (love2hug.com, Feb 2026):** Lovable's SPA hosting serves the React app's catch-all route (`*` → NotFound component) for **any URL that doesn't match a real file with an extension**. This means:
> - `/seo-test/` → SPA 404 (catch-all intercepts before directory index resolution)
> - `/seo-test/index.html` → static file served correctly (has explicit extension)
>
> The `public/subfolder/index.html` pattern works for local dev servers and most traditional hosts, but **not on Lovable hosting**. Use the flat file approach (Option C) or Cloudflare Worker (Part 2, Option C) instead.

### Option C: Flat file + JS redirect (quick fix for Lovable)

**How it works:** Rename your static page to a flat file with an extension (e.g., `public/home.html`). Then add a JS redirect in your React Index component so human visitors at `/` get sent to the static page. Bots don't execute JS, so they still see the empty SPA — you'll need a Cloudflare Worker (Part 2, Option C) for bot coverage.

```
Human at /  → SPA loads → JS redirect → /home.html (static page)
Bot at /    → sees empty <div id="root"> (JS redirect not executed)
```

**Steps:**

1. **Move or copy the static HTML to a flat file:**
   ```
   public/home.html    ← accessible at /home.html (Lovable serves files with extensions)
   ```

2. **Add redirect to your React Index component:**
   ```tsx
   // In your Index page component (e.g., src/pages/Index.tsx)
   import { useEffect } from "react";

   const Index = () => {
     useEffect(() => {
       window.location.replace("/home.html");
     }, []);

     // Minimal fallback while redirect happens
     return null;
   };

   export default Index;
   ```

3. **Verify:**
   ```bash
   # Static file accessible directly
   curl -s https://yoursite.com/home.html | grep "<h1>"

   # Root redirects (browsers follow, bots don't)
   curl -s -L https://yoursite.com/ | grep "<h1>"
   ```

**Caveats:**
- **Redirect flash.** Humans see a brief blank page while the SPA loads and redirects. This is a UX hit (two page loads).
- **Bots are NOT covered.** The JS redirect requires JavaScript execution. Crawlers see the empty SPA, not the static page. **You must pair this with a Cloudflare Worker (Part 2, Option C) for SEO.**
- **Canonical URL.** Set `<link rel="canonical" href="https://yoursite.com/" />` in `home.html` so Google treats `/` as the authoritative URL, not `/home.html`.

**When to use this:** As a quick stopgap to get humans to the static page immediately while you set up the Cloudflare Worker for bots. The Worker alone (without the redirect) is the cleaner long-term solution — see the recommendation below.

> **Recommendation:** If the SPA renders fine for humans and the only issue is bots can't see content, **skip the redirect entirely** and just use the Cloudflare Worker (Part 2, Option C). The Worker serves static HTML to bots at `/` while humans get the normal SPA — no redirect flash, no React code changes, no extra round trip. Only add the `/home.html` redirect if you have a specific reason to send humans to the static page (e.g., the SPA home page is slow, broken, or you want the fastest possible first paint).

---

# Part 2: Technical Path

**Requires:** CLI access, and/or a Cloudflare account.

These options automate or enhance what you proved works in Part 1. **Do Part 1 first** — if `/seo-test/` doesn't show content to crawlers, these won't either, and you'll have wasted time on infrastructure.

---

## Option A: SEO Capture Tool (Experimental Alpha)

A single command that captures a rendered React/Vite SPA and produces a **self-contained static HTML file** ready for search engines and AI crawlers.

```bash
node scripts/capture-spa.mjs https://yoursite.com
```

Or with the full wrapper (capture + local preview):

```bash
./scripts/capture-and-verify.sh https://yoursite.com
```

### What it does (7 steps, fully automated)

| Step | Action | Why |
|------|--------|-----|
| 1 | Navigate to URL | Loads the SPA in headless Chrome |
| 2 | Wait for content | Ensures React has finished rendering |
| 3 | Inline all external CSS | Puppeteer's `page.content()` only captures HTML, not CSS files. Without this, Tailwind utility classes render as unstyled text. This was the #1 bug we hit (see Lessons Learned). |
| 4 | Remove JS module scripts | Crawlers don't execute JavaScript — that's the whole point of this tool |
| 5 | Validate SEO meta tags | Checks for title, description, OG tags, Twitter cards, JSON-LD, canonical URL, h1. Warns about anything missing. |
| 6 | Add capture marker | Injects `<meta name="capture-status">` and timestamp for verification |
| 7 | Write + self-verify | Saves the file, then re-reads it and runs 7 automated checks |

### Setup

```bash
# From the project root
npm install          # installs Puppeteer
```

> **Don't have the scripts yet?** Tell your AI assistant: *"Extract the SEO Capture Tool scripts from STATIC-HOME-PAGE.md into my project."* See the LLM Instructions section at the top of this doc.

### Usage

**Capture from a live site:**
```bash
node scripts/capture-spa.mjs https://yoursite.com
```

**Capture from a local dev server:**
```bash
# Terminal 1: start your Vite dev server
npm run dev

# Terminal 2: capture it
node scripts/capture-spa.mjs http://localhost:5173
```

**Full flow with preview (capture + local server + browser):**
```bash
./scripts/capture-and-verify.sh https://yoursite.com
```

### CLI options

```
node scripts/capture-spa.mjs <url> [options]

Options:
  --output <path>   Output file path (default: public/seo-test/index.html)
  --open            Open the result in the default browser after capture
```

### Example output

```
  ======================================
  SEO Capture Tool — Experimental Alpha
  ======================================
  Source: https://love2hug.dev
  Output: /path/to/public/seo-test/index.html

  [1/7] Navigating to https://love2hug.dev
    PASS  Page loaded (networkidle0)

  [2/7] Waiting for content to render
    PASS  Content element found

  [3/7] Inlining external CSS
    PASS  Inlined 1 stylesheet(s)

  [4/7] Removing module scripts (not needed for SEO)
    PASS  Removed 1 script(s)

  [5/7] Validating SEO meta tags
    PASS  <title>: Love2Hug - AGENTS.md Architecture Guide for Lovable P...
    PASS  meta description: Open-source checklist-driven architecture guide fo...
    PASS  canonical URL: https://love2hug.dev/
    PASS  og:title: Love2Hug - AGENTS.md Architecture Guide for Lovable P...
    PASS  og:description: Open-source checklist-driven architecture guide fo...
    PASS  og:image: https://love2hug.dev/og-image.png
    PASS  og:url: https://love2hug.dev/
    PASS  twitter:card: summary_large_image
    PASS  JSON-LD structured data: present
    PASS  <h1> heading: Make your Lovable code Huggable for users — built for ...

  [6/7] Capturing final HTML
    PASS  Captured 42 KB of HTML

  [7/7] Writing output and verifying
    ..    Written to: /path/to/public/seo-test/index.html
    PASS  Has <h1> content
    PASS  Capture marker present
    PASS  No external stylesheet links (CSS inlined)
    PASS  No module script tags (JS stripped)
    PASS  Has <title> tag
    PASS  Has meta description
    PASS  File size: 42 KB

  ======================================
  Summary
  ======================================
  CSS stylesheets inlined: 1
  JS scripts removed:     1
  SEO tags found:         10/10
  Verification:           ALL PASSED
  Output:                 /path/to/public/seo-test/index.html
```

### How it fits into the workflow

```
1. Run capture tool  →  self-contained /seo-test/index.html
2. Deploy to Lovable  →  verify at https://yoursite.com/seo-test/
3. Check with crawlers  →  curl, Google Rich Results Test, Free SEO Check
4. Promote to /  →  Cloudflare Worker (recommended) or manual copy
```

### Known limitations (Alpha)

- **Requires the target site to be running.** Cannot capture from build output alone (use Option B below for that).
- **CSS inlining fetches from the same origin.** Cross-origin stylesheets (Google Fonts, CDNs) will be inlined if CORS allows, but may fail silently if not.
- **No image optimization.** Images are left as-is (external URLs). For a fully self-contained page, you'd need to also download and embed images.
- **Interactive components won't work.** Accordions, modals, and other JS-driven UI will be frozen in their initial state (collapsed, closed, etc.). This is expected — crawlers don't interact with the page.
- **Color values for custom themes are not extracted automatically.** If you use the Tailwind CDN fallback, you'll need to manually define CSS custom properties. The tool captures whatever CSS the site already has.

---

## Option B: Prerender at Build Time (Puppeteer Postbuild)

**Goal:** Automatically generate a static HTML snapshot of your home page after every build.

**Requires:** CLI access (clone repo from Lovable, run locally or in CI). Puppeteer will **not** work in Lovable's build environment.

### How it works

After `vite build` produces your SPA in `dist/`, a postbuild script launches a headless browser, loads the app, waits for React to render, and saves the output as a static HTML file.

> **Watch out: Puppeteer captures HTML, not CSS.** Puppeteer's `page.content()` returns the DOM as a string but does **not** bundle external stylesheets. If your Vite build splits CSS into `/assets/*.css` files (which it does by default), the captured HTML will reference those files — but they won't exist when the page is served standalone. The result: every Tailwind utility class renders as unstyled raw HTML. The script below fixes this by inlining all `<link rel="stylesheet">` tags into `<style>` blocks before capturing. If you write your own capture script, **always inline CSS first.**

**Key change from the old approach:** We write the output to `dist/seo-test/index.html` instead of overwriting `dist/index.html`. This keeps the SPA intact and gives you a safe prerendered page to verify before promoting to `/`.

### Setup

#### 1. Install Puppeteer

```bash
npm install --save-dev puppeteer
```

#### 2. Create `scripts/prerender-home.mjs`

```javascript
import puppeteer from 'puppeteer';
import { readFile, writeFile, mkdir } from 'fs/promises';
import { resolve } from 'path';
import { createServer } from 'http';

const DIST = resolve('dist');
const PORT = 4173;
const OUTPUT_PATH = 'seo-test/index.html'; // Safe output — doesn't touch the SPA

async function serve() {
  const handler = async (req, res) => {
    const url = req.url === '/' ? '/index.html' : req.url;
    try {
      const file = await readFile(resolve(DIST, `.${url}`));
      const ext = url.split('.').pop();
      const types = {
        html: 'text/html',
        js: 'application/javascript',
        css: 'text/css',
        json: 'application/json',
      };
      res.writeHead(200, { 'Content-Type': types[ext] || 'application/octet-stream' });
      res.end(file);
    } catch {
      const index = await readFile(resolve(DIST, 'index.html'));
      res.writeHead(200, { 'Content-Type': 'text/html' });
      res.end(index);
    }
  };
  const server = createServer(handler);
  return new Promise(r => server.listen(PORT, () => r(server)));
}

async function prerender() {
  console.log('Prerendering home page to /seo-test/ ...');
  const server = await serve();

  const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
  const page = await browser.newPage();

  await page.goto(`http://localhost:${PORT}/`, {
    waitUntil: 'networkidle0',
    timeout: 30000,
  });

  // Wait for React to render — adjust selector for your app
  await page.waitForSelector('h1, [data-testid="home"], main', { timeout: 10000 }).catch(() => {
    console.warn('Warning: Could not find expected home page element. Proceeding anyway.');
  });

  // CRITICAL: Inline all external CSS before capturing HTML.
  // Puppeteer's page.content() only captures the DOM — it does NOT
  // bundle external stylesheets. Without this step, the output HTML
  // references /assets/*.css files that won't exist in the static page,
  // and every Tailwind utility class renders as unstyled raw HTML.
  await page.evaluate(async () => {
    const links = document.querySelectorAll('link[rel="stylesheet"]');
    for (const link of links) {
      try {
        const res = await fetch(link.href);
        const css = await res.text();
        const style = document.createElement('style');
        style.textContent = css;
        link.replaceWith(style);
      } catch (e) {
        console.warn('Could not inline stylesheet:', link.href);
      }
    }
    // Also remove module scripts — JS is not needed for SEO pages
    document.querySelectorAll('script[type="module"]').forEach(s => s.remove());
  });

  const html = await page.content();

  // Add a marker so we can verify prerendering worked
  const marked = html.replace(
    '</head>',
    '<meta name="prerender-status" content="prerendered" />\n</head>'
  );

  // Write to seo-test/ subdirectory — not the SPA root
  const outDir = resolve(DIST, 'seo-test');
  await mkdir(outDir, { recursive: true });
  await writeFile(resolve(outDir, 'index.html'), marked, 'utf-8');
  console.log('Done: dist/seo-test/index.html written with prerendered content.');

  await browser.close();
  server.close();
}

prerender().catch(err => {
  console.error('Prerender failed:', err.message);
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

#### 4. Verify

```bash
npm run build

# Check the capture marker exists
grep -c "prerender-status" dist/seo-test/index.html  # should print 1

# Check the output has real content
grep -c "<h1>" dist/seo-test/index.html  # should print 1+

# IMPORTANT: Check that CSS was inlined (not left as external <link> tags).
# If this prints 0, the CSS was inlined correctly and the page is self-contained.
# If this prints 1+, the page still references external stylesheets and will
# render as unstyled HTML when served standalone.
grep -c 'link rel="stylesheet"' dist/seo-test/index.html  # should print 0

# Quick visual check — serve the file and open in a browser
npx serve dist -l 4173
# Visit http://localhost:4173/seo-test/ — should look styled, not raw HTML
```

### Promoting to `/`

Once you've confirmed `/seo-test/` works, you can optionally copy the output to overwrite `dist/index.html` in the same script. Add this after the `writeFile` call:

```javascript
// Optional: also overwrite the SPA's index.html for the root route
// await writeFile(resolve(DIST, 'index.html'), marked, 'utf-8');
```

Uncomment when you're confident.

---

## Option C: Cloudflare Worker Prerender Proxy (Recommended for Lovable)

**Goal:** Intercept crawler requests at the edge and serve static HTML, without changing your Lovable build at all.

**Requires:** Free Cloudflare account. Your domain must use Cloudflare DNS (or use a `yoursite.workers.dev` subdomain for testing).

**Why this is the recommended approach for Lovable:** The Worker solves both the SEO problem (bots see static HTML) and the Lovable hosting constraint (directory indexes don't resolve — see Part 1, Step 3). No React code changes, no redirect flash, no flat file workarounds. The Worker intercepts at the CDN edge before the request ever hits Lovable's SPA routing.

### How it works

A Cloudflare Worker sits in front of your Lovable site. When it detects a bot user-agent, it serves a prerendered HTML page. Human visitors get the normal SPA.

```
Bot request  → Cloudflare Worker → serves cached static HTML (never hits Lovable)
Human request → Cloudflare Worker → passes through to Lovable SPA (transparent)
```

### Prerequisites

Before setting up the Worker, you need **two things ready**:

1. **A Cloudflare account with your domain.** Your domain must use Cloudflare DNS. If it doesn't yet, add your domain to Cloudflare and update your registrar's nameservers. This is free. You can also test with a `yoursite.workers.dev` subdomain first (no domain transfer needed).

2. **A self-contained static HTML page.** This is the HTML the Worker will serve to bots. It must be fully self-contained — all CSS inlined, no external stylesheet `<link>` tags, no module `<script>` tags. The page should include all your SEO meta tags (title, description, OG, Twitter, JSON-LD, canonical, h1).

**How to get your static HTML:**

| Method | When to use | How |
|--------|-------------|-----|
| **SEO Capture Tool** (recommended) | You have CLI access and a running site | Run `node scripts/capture-spa.mjs https://yoursite.com` — it captures, inlines CSS, strips JS, and validates SEO tags automatically. Output is at `public/seo-test/index.html`. See [Option A](#option-a-seo-capture-tool-experimental-alpha). |
| **Manual from Part 1** | No CLI access, or you wrote the page by hand | Use your verified `/seo-test/index.html` from [Step 1](#step-1-create-your-seo-test-page). Open the file and copy its full HTML. |
| **View Source in browser** | Quick and dirty | Visit your live SPA, View Source (Ctrl+U), copy the full HTML. Then manually add SEO meta tags and inline/remove CSS/JS references. Not recommended — easy to miss things. |

**Verify your HTML is ready before proceeding:**

```bash
# Has real content (not an empty div)?
grep -c "<h1>" public/seo-test/index.html          # should print 1+

# No external stylesheets (CSS must be inlined)?
grep -c 'link rel="stylesheet"' public/seo-test/index.html  # should print 0

# No module scripts (JS must be stripped)?
grep -c 'type="module"' public/seo-test/index.html  # should print 0

# Has SEO meta tags?
grep -c "og:title" public/seo-test/index.html       # should print 1+
```

If any of these fail, go back and fix your HTML first. The Worker will serve exactly what you give it — garbage in, garbage out.

### Setup

#### 1. Find your Lovable origin URL

You'll need your Lovable project's direct hosting URL (not your custom domain). This is the URL the Worker will proxy non-bot requests to. Find it in your Lovable project settings — it looks like `https://your-project-id.lovable.app`.

#### 2. Create the Worker

In Cloudflare dashboard → Workers & Pages → Create Worker:

```javascript
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

// Paste your verified /seo-test/ HTML here
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
  <!-- Your verified content from /seo-test/ goes here -->
  <main>
    <h1>Your Headline</h1>
    <p>Your content that crawlers need to see.</p>
  </main>
</body>
</html>`;

// IMPORTANT: Replace this with your actual Lovable hosting URL
const ORIGIN = 'https://your-project-id.lovable.app';

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

    // Everything else: fetch from the actual Lovable origin
    // NOTE: Do NOT use fetch(request) — that creates an infinite loop
    // because the Worker intercepts its own request. Fetch from the
    // origin URL explicitly instead.
    const originUrl = new URL(url.pathname + url.search, ORIGIN);
    return fetch(originUrl, {
      method: request.method,
      headers: request.headers,
    });
  },
};
```

#### 3. Paste your static HTML into the Worker

In the Worker code above, replace the placeholder `PRERENDERED_HOME` string with your actual verified HTML. Copy the entire contents of your `public/seo-test/index.html` (or wherever your static page lives) and paste it between the backticks.

Also replace `https://your-project-id.lovable.app` with your actual Lovable origin URL from step 1.

#### 4. Route the Worker

In Cloudflare dashboard:

1. Go to **Websites** → select your domain
2. Go to **Workers Routes** (under the Workers tab)
3. Add a route: `yoursite.com/*` → select your Worker
4. Make sure your domain's DNS is proxied (orange cloud icon) through Cloudflare

**If you're testing with a `workers.dev` subdomain first:** Skip this step — the Worker is already accessible at `your-worker-name.your-account.workers.dev`. Test with that URL before pointing your real domain.

#### 5. Verify

```bash
# Simulate Googlebot — should return your static HTML with real content
curl -s -A "Googlebot" https://yoursite.com/ | grep "<h1>"

# Simulate ChatGPT — should also return static HTML
curl -s -A "GPTBot" https://yoursite.com/ | grep "<h1>"

# Normal browser request — should still return the SPA (empty div)
curl -s https://yoursite.com/ | grep '<div id="root">'

# Check the prerender header is present
curl -sI -A "Googlebot" https://yoursite.com/ | grep -i "x-prerender"
```

**All four should pass.** If the bot requests don't return your content, check that:
- Your Worker route matches the domain exactly
- The `PRERENDERED_HOME` string contains your actual HTML (not the placeholder)
- Your DNS is proxied through Cloudflare (not DNS-only)

### Automating snapshot refresh

The hardcoded HTML string is manual — same problem as the Spike approach. To automate:

- **Cloudflare KV + Cron Trigger:** Store the HTML in KV. A scheduled trigger runs weekly, fetches your live `/seo-test/` page, and updates the KV value.
- **GitHub Action:** A cron job uses Puppeteer to render your home page and updates the Worker script via the Cloudflare API.

---

## Decision flowchart

```
Start here
│
├── Create public/seo-test/index.html (Part 1 — everyone does this)
│     └── Can you access the file?
│           ├── /seo-test/ returns 404 → Normal on Lovable (directory indexes don't resolve)
│           │     └── Try /seo-test/index.html (explicit extension) — does THAT work?
│           │           ├── No → Check file path, redeploy
│           │           └── Yes → File exists, you just need a serving strategy ↓
│           └── /seo-test/ works → Great, continue ↓
│
├── Is manual updates to /seo-test/ enough for you?
│     └── Yes → Done. Update the HTML when your content changes.
│
├── Want to automate the capture?
│     ├── Have CLI access? → Option A (SEO Capture Tool) or Option B (Puppeteer postbuild)
│     └── No CLI? → Skip to promotion step below
│
└── Want bots to see this content on / instead of /seo-test/?
      ├── Quick fix: Flat file /home.html + JS redirect (covers humans, NOT bots)
      ├── Recommended: Cloudflare Worker at / for bots (covers bots, clean)
      └── Both: Redirect for humans + Worker for bots (full coverage)
```

---

# Embedded Scripts (Source of Truth)

> **LLM Instructions:** When extracting these scripts, copy the code exactly as shown. Create the `scripts/` directory if it doesn't exist. Make `capture-and-verify.sh` executable (`chmod +x`). Merge the `"scripts"` and `"devDependencies"` entries into the project's existing `package.json` — do not overwrite other fields. Run `npm install` after updating `package.json`.

## Script 1: `scripts/capture-spa.mjs`

Core Node.js script — does all 7 capture steps.

**Depends on:** `puppeteer` (devDependency)

```javascript
#!/usr/bin/env node

/**
 * capture-spa.mjs — Capture a rendered SPA as a self-contained static HTML page
 *
 * EXPERIMENTAL ALPHA — Part of the SEO Capture Tool (see OFF-ROAD/SEO-CAPTURE-TOOL.md)
 *
 * Usage:
 *   node scripts/capture-spa.mjs <url> [--output <path>] [--open]
 *
 * Examples:
 *   node scripts/capture-spa.mjs https://love2hug.dev
 *   node scripts/capture-spa.mjs http://localhost:5173 --output dist/seo-test/index.html
 *   node scripts/capture-spa.mjs https://yoursite.com --open
 *
 * What it does:
 *   1. Launches headless Chrome via Puppeteer
 *   2. Navigates to the URL, waits for content to render
 *   3. Inlines all external CSS into <style> blocks
 *   4. Removes <script type="module"> tags (not needed for SEO)
 *   5. Validates SEO meta tags and warns about missing ones
 *   6. Adds a capture marker for verification
 *   7. Writes the self-contained HTML to public/seo-test/index.html
 *   8. Runs verification checks on the output
 */

import puppeteer from 'puppeteer';
import { readFile, writeFile, mkdir } from 'fs/promises';
import { resolve } from 'path';

// --- Configuration ---
const DEFAULTS = {
  output: resolve('public', 'seo-test', 'index.html'),
  timeout: 30000,
  contentTimeout: 10000,
  contentSelectors: 'h1, [data-testid="home"], main, [role="main"]',
};

// --- CLI argument parsing ---
function parseArgs(argv) {
  const args = argv.slice(2);
  const config = { url: null, output: DEFAULTS.output, open: false };

  for (let i = 0; i < args.length; i++) {
    if (args[i] === '--output' && args[i + 1]) {
      config.output = resolve(args[++i]);
    } else if (args[i] === '--open') {
      config.open = true;
    } else if (!args[i].startsWith('--')) {
      config.url = args[i];
    }
  }
  return config;
}

// --- Logging helpers ---
const log = {
  step: (n, msg) => console.log(`\n  [${n}/7] ${msg}`),
  pass: (msg) => console.log(`    PASS  ${msg}`),
  warn: (msg) => console.log(`    WARN  ${msg}`),
  fail: (msg) => console.log(`    FAIL  ${msg}`),
  info: (msg) => console.log(`    ..    ${msg}`),
};

// --- Step 1: Navigate ---
async function navigate(page, url) {
  log.step(1, `Navigating to ${url}`);
  try {
    await page.goto(url, { waitUntil: 'networkidle0', timeout: DEFAULTS.timeout });
    log.pass('Page loaded (networkidle0)');
  } catch (e) {
    log.fail(`Navigation failed: ${e.message}`);
    throw e;
  }
}

// --- Step 2: Wait for content ---
async function waitForContent(page) {
  log.step(2, 'Waiting for content to render');
  try {
    await page.waitForSelector(DEFAULTS.contentSelectors, { timeout: DEFAULTS.contentTimeout });
    log.pass('Content element found');
  } catch {
    log.warn('No h1/main/[data-testid="home"] found — page may be empty or use different selectors');
  }
}

// --- Step 3: Inline CSS ---
async function inlineCSS(page) {
  log.step(3, 'Inlining external CSS');
  const result = await page.evaluate(async () => {
    const links = document.querySelectorAll('link[rel="stylesheet"]');
    let inlined = 0;
    let failed = 0;
    for (const link of links) {
      try {
        const res = await fetch(link.href);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const css = await res.text();
        const style = document.createElement('style');
        style.setAttribute('data-inlined-from', link.getAttribute('href') || 'unknown');
        style.textContent = css;
        link.replaceWith(style);
        inlined++;
      } catch {
        failed++;
      }
    }
    return { total: links.length, inlined, failed };
  });

  if (result.total === 0) {
    log.info('No external stylesheets found (CSS may already be inline)');
  } else if (result.failed > 0) {
    log.warn(`Inlined ${result.inlined}/${result.total} stylesheets (${result.failed} failed)`);
  } else {
    log.pass(`Inlined ${result.inlined} stylesheet(s)`);
  }
  return result;
}

// --- Step 4: Strip JS ---
async function stripJS(page) {
  log.step(4, 'Removing module scripts (not needed for SEO)');
  const removed = await page.evaluate(() => {
    const scripts = document.querySelectorAll('script[type="module"], script[src*="/assets/"]');
    const count = scripts.length;
    scripts.forEach(s => s.remove());
    return count;
  });

  if (removed > 0) {
    log.pass(`Removed ${removed} script(s)`);
  } else {
    log.info('No module/asset scripts found');
  }
  return removed;
}

// --- Step 5: Validate SEO tags ---
async function validateSEO(page) {
  log.step(5, 'Validating SEO meta tags');
  const seo = await page.evaluate(() => {
    const get = (sel) => {
      const el = document.querySelector(sel);
      return el ? (el.getAttribute('content') || el.textContent || '').trim() : null;
    };
    return {
      title: document.title || null,
      description: get('meta[name="description"]'),
      canonical: document.querySelector('link[rel="canonical"]')?.getAttribute('href') || null,
      ogTitle: get('meta[property="og:title"]'),
      ogDescription: get('meta[property="og:description"]'),
      ogImage: get('meta[property="og:image"]'),
      ogUrl: get('meta[property="og:url"]'),
      twitterCard: get('meta[name="twitter:card"]'),
      jsonLd: document.querySelector('script[type="application/ld+json"]') ? 'present' : null,
      h1: document.querySelector('h1')?.textContent?.trim()?.substring(0, 80) || null,
    };
  });

  const checks = [
    ['<title>', seo.title],
    ['meta description', seo.description],
    ['canonical URL', seo.canonical],
    ['og:title', seo.ogTitle],
    ['og:description', seo.ogDescription],
    ['og:image', seo.ogImage],
    ['og:url', seo.ogUrl],
    ['twitter:card', seo.twitterCard],
    ['JSON-LD structured data', seo.jsonLd],
    ['<h1> heading', seo.h1],
  ];

  let passed = 0;
  let warned = 0;
  for (const [name, value] of checks) {
    if (value) {
      log.pass(`${name}: ${value.substring(0, 60)}${value.length > 60 ? '...' : ''}`);
      passed++;
    } else {
      log.warn(`${name}: MISSING`);
      warned++;
    }
  }

  return { seo, passed, warned, total: checks.length };
}

// --- Step 6: Capture + marker ---
async function captureHTML(page) {
  log.step(6, 'Capturing final HTML');
  const html = await page.content();
  const timestamp = new Date().toISOString();
  const marked = html.replace(
    '</head>',
    `<meta name="capture-status" content="captured" />\n<meta name="capture-timestamp" content="${timestamp}" />\n</head>`
  );
  log.pass(`Captured ${(marked.length / 1024).toFixed(0)} KB of HTML`);
  return marked;
}

// --- Step 7: Write + verify ---
async function writeAndVerify(html, outputPath) {
  log.step(7, 'Writing output and verifying');

  const outDir = resolve(outputPath, '..');
  await mkdir(outDir, { recursive: true });
  await writeFile(outputPath, html, 'utf-8');
  log.info(`Written to: ${outputPath}`);

  // Verification checks on the written file
  const content = await readFile(outputPath, 'utf-8');
  const checks = {
    hasContent: /<h1[^>]*>/.test(content),
    hasMarker: content.includes('capture-status'),
    noExternalCSS: !/<link[^>]*rel=["']stylesheet["'][^>]*>/.test(content),
    noModuleScripts: !/<script[^>]*type=["']module["'][^>]*>/.test(content),
    hasTitle: /<title[^>]*>[^<]+<\/title>/.test(content),
    hasDescription: /meta[^>]*name=["']description["']/.test(content),
    sizeOK: content.length > 1000,
  };

  const labels = {
    hasContent: 'Has <h1> content',
    hasMarker: 'Capture marker present',
    noExternalCSS: 'No external stylesheet links (CSS inlined)',
    noModuleScripts: 'No module script tags (JS stripped)',
    hasTitle: 'Has <title> tag',
    hasDescription: 'Has meta description',
    sizeOK: `File size: ${(content.length / 1024).toFixed(0)} KB`,
  };

  let allPassed = true;
  for (const [key, passed] of Object.entries(checks)) {
    if (passed) {
      log.pass(labels[key]);
    } else {
      log.fail(labels[key]);
      allPassed = false;
    }
  }

  return { allPassed, checks };
}

// --- Main ---
async function main() {
  const config = parseArgs(process.argv);

  if (!config.url) {
    console.error(`
  SEO Capture Tool — Experimental Alpha

  Usage: node scripts/capture-spa.mjs <url> [options]

  Options:
    --output <path>   Output file path (default: public/seo-test/index.html)
    --open            Open the result in the default browser after capture

  Examples:
    node scripts/capture-spa.mjs https://yoursite.com
    node scripts/capture-spa.mjs http://localhost:5173 --open
    node scripts/capture-spa.mjs https://yoursite.com --output dist/seo-test/index.html
`);
    process.exit(1);
  }

  console.log('\n  ======================================');
  console.log('  SEO Capture Tool — Experimental Alpha');
  console.log('  ======================================');
  console.log(`  Source: ${config.url}`);
  console.log(`  Output: ${config.output}`);

  const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });

  try {
    const page = await browser.newPage();
    // Set a realistic viewport
    await page.setViewport({ width: 1280, height: 800 });

    await navigate(page, config.url);
    await waitForContent(page);
    const cssResult = await inlineCSS(page);
    const jsRemoved = await stripJS(page);
    const seoResult = await validateSEO(page);
    const html = await captureHTML(page);
    const verifyResult = await writeAndVerify(html, config.output);

    // --- Summary ---
    console.log('\n  ======================================');
    console.log('  Summary');
    console.log('  ======================================');
    console.log(`  CSS stylesheets inlined: ${cssResult.inlined}`);
    console.log(`  JS scripts removed:     ${jsRemoved}`);
    console.log(`  SEO tags found:         ${seoResult.passed}/${seoResult.total}`);
    console.log(`  Verification:           ${verifyResult.allPassed ? 'ALL PASSED' : 'SOME CHECKS FAILED'}`);
    console.log(`  Output:                 ${config.output}`);

    if (seoResult.warned > 0) {
      console.log(`\n  ${seoResult.warned} SEO tag(s) missing — see warnings above.`);
    }

    if (!verifyResult.allPassed) {
      console.log('\n  Some verification checks failed. Review the output before deploying.');
    }

    console.log('');

    // Open in browser if requested
    if (config.open) {
      const { exec } = await import('child_process');
      const cmd = process.platform === 'darwin' ? 'open' : process.platform === 'win32' ? 'start' : 'xdg-open';
      exec(`${cmd} "${config.output}"`);
      console.log(`  Opened in browser.\n`);
    }

    process.exit(verifyResult.allPassed ? 0 : 1);
  } finally {
    await browser.close();
  }
}

main().catch(err => {
  console.error(`\n  FATAL: ${err.message}\n`);
  process.exit(1);
});
```

## Script 2: `scripts/capture-and-verify.sh`

Bash wrapper — runs the Node script, starts a local preview server, opens your browser.

**Depends on:** Node.js, Puppeteer (npm), Python 3 (optional, for local preview)

```bash
#!/usr/bin/env bash
#
# capture-and-verify.sh — End-to-end SEO capture + local preview
#
# EXPERIMENTAL ALPHA — Part of the SEO Capture Tool
#
# Usage:
#   ./scripts/capture-and-verify.sh <url>
#   ./scripts/capture-and-verify.sh https://yoursite.com
#   ./scripts/capture-and-verify.sh http://localhost:5173
#
# What it does:
#   1. Runs the Node capture script (capture-spa.mjs)
#   2. Starts a local Python HTTP server
#   3. Opens the captured page in your browser
#   4. Waits for you to review, then cleans up
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
OUTPUT_DIR="$PROJECT_DIR/public"
OUTPUT_FILE="$OUTPUT_DIR/seo-test/index.html"
SERVER_PORT=0  # will be assigned

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
CYAN='\033[0;36m'
NC='\033[0m' # No Color
BOLD='\033[1m'

# --- Helpers ---
info()  { echo -e "${CYAN}[info]${NC}  $1"; }
ok()    { echo -e "${GREEN}[ok]${NC}    $1"; }
warn()  { echo -e "${YELLOW}[warn]${NC}  $1"; }
fail()  { echo -e "${RED}[fail]${NC}  $1"; }

cleanup() {
  if [ -n "${SERVER_PID:-}" ] && kill -0 "$SERVER_PID" 2>/dev/null; then
    kill "$SERVER_PID" 2>/dev/null || true
    info "Stopped local server (PID $SERVER_PID)"
  fi
}
trap cleanup EXIT

# --- Argument check ---
if [ $# -lt 1 ]; then
  echo ""
  echo -e "${BOLD}  SEO Capture Tool — Experimental Alpha${NC}"
  echo ""
  echo "  Usage: $0 <url>"
  echo ""
  echo "  Examples:"
  echo "    $0 https://yoursite.com"
  echo "    $0 http://localhost:5173"
  echo ""
  exit 1
fi

URL="$1"

echo ""
echo -e "${BOLD}  ======================================${NC}"
echo -e "${BOLD}  SEO Capture + Verify${NC}"
echo -e "${BOLD}  ======================================${NC}"
echo ""

# --- Step 1: Check dependencies ---
info "Checking dependencies..."

if ! command -v node &>/dev/null; then
  fail "Node.js not found. Install it from https://nodejs.org"
  exit 1
fi
ok "Node.js $(node --version)"

if ! [ -d "$PROJECT_DIR/node_modules/puppeteer" ]; then
  warn "Puppeteer not installed. Running npm install..."
  (cd "$PROJECT_DIR" && npm install)
fi
ok "Puppeteer installed"

if ! command -v python3 &>/dev/null; then
  warn "Python3 not found — skipping local preview server"
  NO_SERVER=true
else
  ok "Python3 $(python3 --version 2>&1 | awk '{print $2}')"
  NO_SERVER=false
fi

# --- Step 2: Run capture ---
echo ""
info "Running capture script..."
echo ""

node "$SCRIPT_DIR/capture-spa.mjs" "$URL" --output "$OUTPUT_FILE"
CAPTURE_EXIT=$?

if [ $CAPTURE_EXIT -ne 0 ]; then
  echo ""
  fail "Capture script exited with errors (code $CAPTURE_EXIT)"
  echo "  Review the warnings above and fix any issues before deploying."
  echo ""
  exit $CAPTURE_EXIT
fi

# --- Step 3: Start local server + open browser ---
if [ "$NO_SERVER" = false ] && [ -f "$OUTPUT_FILE" ]; then
  echo ""
  info "Starting local preview server..."

  # Find an available port
  SERVER_PORT=$(python3 -c "import socket; s=socket.socket(); s.bind(('',0)); print(s.getsockname()[1]); s.close()")

  (cd "$OUTPUT_DIR" && python3 -m http.server "$SERVER_PORT" --bind 127.0.0.1 &>/dev/null) &
  SERVER_PID=$!
  sleep 1

  PREVIEW_URL="http://localhost:$SERVER_PORT/seo-test/"

  if kill -0 "$SERVER_PID" 2>/dev/null; then
    ok "Server running at $PREVIEW_URL"

    # Open in browser
    if [ "$(uname)" = "Darwin" ]; then
      open "$PREVIEW_URL"
    elif command -v xdg-open &>/dev/null; then
      xdg-open "$PREVIEW_URL"
    fi

    echo ""
    echo -e "  ${BOLD}Preview is open in your browser.${NC}"
    echo "  Press Enter to stop the server and exit..."
    read -r
  else
    warn "Server failed to start on port $SERVER_PORT"
  fi
fi

echo ""
ok "Done. Output is at: $OUTPUT_FILE"
echo ""
```

## `package.json` additions

Merge these into your project's `package.json`:

```json
{
  "scripts": {
    "capture": "node scripts/capture-spa.mjs",
    "capture:verify": "./scripts/capture-and-verify.sh"
  },
  "devDependencies": {
    "puppeteer": "^24.37.3"
  }
}
```

---

## Files checklist

Regardless of which path you use, make sure these files exist:

| File | Purpose | How to verify |
|------|---------|---------------|
| `public/robots.txt` | Tells crawlers what to index | `curl https://yoursite.com/robots.txt` |
| `public/sitemap.xml` | Lists all indexable URLs | `curl https://yoursite.com/sitemap.xml` |
| `public/llms.txt` | Plain-text summary for AI crawlers | `curl https://yoursite.com/llms.txt` |
| `public/seo-test/index.html` | Your verified static SEO page | `curl https://yoursite.com/seo-test/` |

### Minimal `robots.txt`

```
User-agent: *
Allow: /

Sitemap: https://yoursite.com/sitemap.xml
```

**Note:** Lovable does not auto-generate `robots.txt` or `sitemap.xml`. These are fully manual files you create in `public/`. Update `sitemap.xml` whenever you add or remove routes.

### Minimal `sitemap.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://yoursite.com/</loc>
    <lastmod>2026-02-14</lastmod>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://yoursite.com/seo-test/</loc>
    <lastmod>2026-02-14</lastmod>
    <priority>0.8</priority>
  </url>
</urlset>
```

### Minimal `llms.txt`

Since this doc explicitly targets AI crawler visibility, consider adding a `public/llms.txt` file. This is a plain-text file that AI models (ChatGPT, Claude, Perplexity, etc.) look for to understand your site without parsing HTML.

```
# Your Site Name

> One-line description of what your site does.

## About
Your site helps [audience] do [thing]. Founded in [year].

## Key pages
- Home: https://yoursite.com/
- Pricing: https://yoursite.com/pricing
- About: https://yoursite.com/about
- Docs: https://yoursite.com/docs

## Contact
- Email: hello@yoursite.com
- Twitter: @yourhandle
```

Keep it short and factual. This is what AI models read when someone asks "What is YourSite?" — make sure the answer is here in plain text.

---

## Lessons Learned on this Project
Attn LLMs: please update this section with real world results and adaptations.

Live site on this project: http://love2hug.com

### 2026-02-14 — Capture script grabs HTML but not CSS/JS assets

**Problem:** The Puppeteer capture script (`scripts/capture-spa.mjs`) successfully captured the rendered HTML from the live SPA into `public/seo-test/index.html`, but it only saves the HTML document — not the external CSS and JS assets referenced in `<head>`. Empty placeholder files were created at `public/assets/index-Bi-pjKIu.css` (0 bytes) and `public/assets/index-Bi0bFsIi.js` (0 bytes). The captured HTML uses Tailwind CSS utility classes throughout (`flex`, `items-center`, `bg-background`, `text-foreground`, etc.), so without the CSS definitions the page rendered as completely unstyled raw HTML on the local test server.

**Root cause:** Puppeteer's `page.content()` returns the DOM as HTML but does not bundle external stylesheets or scripts. The Vite build output splits CSS and JS into separate hashed asset files. The capture script was never designed to also fetch and save those assets.

**Fix applied:** Replaced the broken external `<script>` and `<link>` references to the empty asset files with:
1. **Tailwind CDN** (`<script src="https://cdn.tailwindcss.com"></script>`) — provides all standard Tailwind utilities at runtime
2. **Inline `tailwind.config`** — maps the site's custom color names (`coral`, `sage`, `cream`, `peach`, `plum`, `amber`, `bg-deep`, `bg-warm`, `text-soft`, `text-muted`, etc.) to HSL CSS custom properties
3. **CSS custom properties in a `<style>` block** — defines `:root` values for `--background`, `--foreground`, `--primary`, `--coral`, `--sage`, and all other theme tokens

This makes the SEO test page fully self-contained, which aligns with the doc's own recommendation in the Step 1 template (inline styles, no external dependencies).

**Takeaway for the capture script:** If you use Puppeteer to capture a Vite/React SPA, you must also handle external assets. Options:
- **Inline CSS during capture:** Use `page.evaluate()` to read all `<style>` and `<link rel="stylesheet">` content and inject it as inline `<style>` blocks before calling `page.content()`
- **Use Tailwind CDN as a fallback:** For Tailwind-based sites, adding the CDN script + theme config is simpler than reconstructing the Vite build output
- **Download assets separately:** Fetch each `/assets/*.css` and `/assets/*.js` URL and write them alongside the HTML

The JS bundle is not needed for an SEO test page (crawlers don't execute JavaScript — that's the whole point). The CSS is critical for visual verification in a browser.

**Color accuracy note:** The HSL values in the CSS custom properties are approximations based on the class names and site branding. To get exact values, inspect the live site's computed styles once it is back online, or extract them from the Lovable project's `src/index.css` or Tailwind config.

### 2026-02-14 — Lovable hosting doesn't resolve directory indexes

**Problem:** `public/seo-test/index.html` exists with valid HTML, but visiting `/seo-test/` or `/seo-test` on Lovable hosting returns the SPA's 404 page (NotFound component). The file IS accessible at `/seo-test/index.html` (with explicit `.html` extension).

**Root cause:** Lovable's SPA hosting serves the React app's catch-all route (`*` → NotFound) for any URL that doesn't match a real file with an extension. The server does not perform directory index resolution (`/folder/` → `/folder/index.html`) before the SPA catch-all kicks in. This is standard SPA hosting behavior (Netlify, Vercel, etc. all have similar routing), but it breaks the common assumption that `public/subfolder/index.html` will be served at `/subfolder/`.

**Workarounds discovered:**
1. **Flat file with extension:** Move `public/seo-test/index.html` to `public/home.html`. Accessible at `/home.html` directly. Combine with a JS redirect from the React Index component for human visitors.
2. **Cloudflare Worker (recommended):** Intercept bot requests at `/` at the CDN edge, before they hit Lovable's SPA routing. Serves static HTML to crawlers while humans get the normal SPA. No React code changes needed.

**Takeaway:** On Lovable (and similar SPA hosts), never rely on directory index resolution for static pages. Either use flat files with explicit extensions (`/page.html`) or handle routing at the edge (Cloudflare Worker, Vercel rewrites, Netlify redirects).

### 2026-02-14 — Abandoned SSG entry point causes build errors

**Problem:** An abandoned `src/main-ssg.tsx` file (SSG entry point) and `ssgOptions` in `vite.config.ts` caused build errors. The project uses standard SPA architecture, not SSG.

**Fix:** Deleted `src/main-ssg.tsx` and removed `ssgOptions` from `vite.config.ts`. If you're following this guide, don't try to add SSG to a Lovable project — it's not compatible with Lovable's build pipeline. Use the capture tool or Cloudflare Worker approach instead.
