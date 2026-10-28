---
name: tool-site-seo-check
description: Audit a small tool/calculator website (Astro or similar static site) for SEO and AdSense readiness before launch or deploy, or when Search Console shows pages not ranking.
---

# Tool site SEO check

Use this on micro-tool websites (calculators, testers, converters) built as static sites, typically Astro deployed to Cloudflare Pages or Vercel. Run it:
- before the first launch of a new site
- before each deploy that adds or changes pages
- when Search Console shows pages stuck, not indexed, or "Alternate page with proper canonical tag"

Audit first, report, then fix only after the user agrees. Never deploy without asking.

## Step 1: Build and run the automated audit

Build the site (e.g. `npm run build`), then run this script against the output folder (`dist/` for Astro, `out/` for a static Next export). Pass the site's production origin.

Save as `seo-audit.mjs` in a scratch folder (not in the repo) and run `node seo-audit.mjs <dist-folder> https://example.com`:

```js
import fs from 'fs'; import path from 'path';
const [dir, origin] = process.argv.slice(2);
const files = [];
(function walk(d){ for (const f of fs.readdirSync(d)) { const p = path.join(d,f); fs.statSync(p).isDirectory() ? walk(p) : p.endsWith('.html') && files.push(p); } })(dir);
const urlFor = f => { let r = '/' + path.relative(dir, f).replace(/\\/g,'/'); r = r.replace(/index\.html$/,'').replace(/\.html$/,''); return origin + (r.length > 1 ? r.replace(/\/$/,'') : '/'); };
const get = (h, re) => (h.match(re) || [])[1];
const titles = {}, descs = {};
const sitemapPath = path.join(dir, 'sitemap.xml');
const sitemap = fs.existsSync(sitemapPath) ? fs.readFileSync(sitemapPath,'utf8') : '';
const smUrls = [...sitemap.matchAll(/<loc>([^<]+)<\/loc>/g)].map(m => m[1].replace(/\/$/,'') || origin);
const rows = [];
for (const f of files) {
  const h = fs.readFileSync(f,'utf8'); const url = urlFor(f); const issues = [];
  const noindex = /<meta[^>]+name="robots"[^>]+noindex/i.test(h);
  const canon = get(h, /<link[^>]+rel="canonical"[^>]+href="([^"]+)"/i);
  const title = get(h, /<title>([^<]*)<\/title>/i) || '';
  const desc = get(h, /<meta[^>]+name="description"[^>]+content="([^"]*)"/i) || '';
  const ogUrl = get(h, /<meta[^>]+property="og:url"[^>]+content="([^"]+)"/i);
  const h1s = (h.match(/<h1[\s>]/gi) || []).length;
  const text = h.replace(/<script[\s\S]*?<\/script>|<style[\s\S]*?<\/style>/gi,' ').replace(/<[^>]+>/g,' ');
  const words = text.split(/\s+/).filter(w => /\p{L}/u.test(w)).length;
  if (!noindex) {
    if (!canon) issues.push('no canonical');
    else if (canon.replace(/\/$/,'') !== url.replace(/\/$/,'')) issues.push(`canonical points to ${canon}`);
    if (ogUrl && ogUrl.replace(/\/$/,'') !== url.replace(/\/$/,'')) issues.push(`og:url is ${ogUrl}`);
    if (title.length < 25 || title.length > 65) issues.push(`title length ${title.length}`);
    if (desc.length < 70 || desc.length > 160) issues.push(`description length ${desc.length}`);
    if (h1s !== 1) issues.push(`${h1s} h1 tags`);
    if (words < 600) issues.push(`thin: ~${words} words`);
    if (!/<html[^>]+lang=/i.test(h)) issues.push('no html lang');
    if (sitemap && !smUrls.includes(url.replace(/\/$/,'') || origin)) issues.push('missing from sitemap');
    (titles[title] ||= []).push(url); (descs[desc] ||= []).push(url);
    for (const m of h.matchAll(/<script type="application\/ld\+json"[^>]*>([\s\S]*?)<\/script>/gi)) { try { JSON.parse(m[1]); } catch { issues.push('invalid JSON-LD'); } }
  }
  rows.push({ url, noindex, issues });
}
for (const [t,u] of Object.entries(titles)) if (u.length > 1) console.log('DUPLICATE TITLE:', t, u);
for (const [d,u] of Object.entries(descs)) if (u.length > 1) console.log('DUPLICATE DESCRIPTION:', u);
const pageUrls = rows.filter(r => !r.noindex).map(r => r.url.replace(/\/$/,'') || origin);
for (const s of smUrls) if (!pageUrls.includes(s)) console.log('SITEMAP URL WITH NO INDEXABLE PAGE:', s);
for (const r of rows) console.log(`${r.issues.length ? 'FAIL' : 'OK  '} ${r.noindex ? '[noindex] ' : ''}${r.url}${r.issues.length ? '\n     - ' + r.issues.join('\n     - ') : ''}`);
```

The word count is rough (it includes nav and footer text); treat it as a signal, not a verdict.

## Step 2: Manual checks the script can't do

**Canonicals and URLs**
- The layout must build the canonical from the current path (in Astro: `new URL(Astro.url.pathname, Astro.site)`), never a hardcoded homepage default. This is the most common and most damaging bug: every page canonicalizing to the homepage hides the whole site.
- Trailing slashes are consistent across sitemap, internal links, canonicals and what the host actually serves. In Astro, prefer `build: { format: 'file' }` with `trailingSlash: 'never'` on Cloudflare Pages.
- Preview and staging domains (`*.pages.dev`, `*.vercel.app`) are noindexed via headers.
- 404 and 500 pages are `noindex`.

**Sitemap and robots**
- `robots.txt` allows crawling and links the sitemap.
- The sitemap lists every indexable page and nothing else (no redirected, noindexed or legal-only pages needed, but they are allowed). Prefer generating it (`@astrojs/sitemap`) over a hand-written file that goes stale.

**Duplicate intent (keyword cannibalization)**
- List each page's target keyword and which tool component it renders. Two pages with the same component and the same search intent compete with each other. Recommend merging with a 301 redirect (`public/_redirects` on Cloudflare Pages, `vercel.json` redirects on Vercel), keeping the URL that best matches the keyword or the domain name.

**Content depth**
- Each tool page should have roughly 800–1,200 words unique to that page: how to use the tool, worked examples (tables help), the rules or formulas behind it, and 5–8 FAQs.
- FAQ schema (`FAQPage`) must match FAQs that are visible on the page.
- Tool schema (`WebApplication`) `url` must be the page's own URL, not the homepage.

**Internal linking**
- Every tool page is reachable from the header navigation or a tools menu, not only the footer.
- Each tool page has a related-tools block linking 3–5 sibling pages.
- Breadcrumbs are visible and have `BreadcrumbList` schema.

**AdSense readiness**
- About, Contact, Privacy policy and Terms pages exist and are linked from the footer (French sites also need Mentions légales).
- No thin or placeholder pages, no broken tools, no lorem ipsum.
- After approval: `ads.txt` at the site root; reserve space for ad slots so they don't cause layout shift.

**Performance and basics**
- Tools work with no console errors; check the browser console on each tool page.
- Images have `alt` text and set width/height; favicon and OG image exist.

## Step 3: Report

Give the user a short report, worst problems first:

1. **Critical** (blocks indexing or ranking): wrong canonicals, noindex on real pages, pages missing from sitemap, duplicate-intent pages.
2. **Important** (holds back ranking or AdSense): thin content, missing internal links, missing legal pages, schema errors.
3. **Nice to have**: title/description lengths, alt text, OG images.

For each item, name the file to change and the exact fix. Then ask which ones to implement. After fixes, rebuild and rerun the script to confirm.

If anything critical was fixed, remind the user after deploy to resubmit the sitemap in Search Console and use URL inspection → Request indexing on the affected pages.
