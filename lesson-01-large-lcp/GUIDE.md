# Lesson 01 — Extremely Large Hero Image

**Metric:** Largest Contentful Paint (LCP)
**Also affects:** First Contentful Paint (FCP), Speed Index

---

## Learning Objectives

By the end of this lesson you should know:

- ✔ Why images so often dominate LCP
- ✔ How browsers discover, prioritize, and download images
- ✔ Why image *dimensions* (not just file size) matter
- ✔ The real tradeoffs between JPEG, WebP, and AVIF
- ✔ What `fetchpriority` actually does and when to use it
- ✔ What `preload` actually does and when it helps vs. hurts
- ✔ How responsive images (`srcset` / `sizes` / `<picture>`) work
- ✔ Why `width` / `height` attributes matter even when CSS also sizes the box
- ✔ How to measure an improvement instead of just assuming one happened

---

## 1. What is this metric?

**Largest Contentful Paint (LCP)** is the render time of the largest image or
text block visible within the viewport, measured from when the page starts
loading.

| Score | Range |
|---|---|
| 🟢 Good | ≤ 2.5s |
| 🟠 Needs Improvement | 2.5s – 4.0s |
| 🔴 Poor | > 4.0s |

LCP is one of Google's three **Core Web Vitals** (alongside CLS and INP), which
means it directly affects both real user experience and, to a degree, search
ranking signals.

---

## 2. How does the browser work internally?

Roughly, in order:

1. **HTML parsing begins.** The browser reads HTML top to bottom, building the DOM.
2. **Preload scanner runs ahead of the main parser**, speculatively looking for
   resources (images, scripts, stylesheets) it can start fetching early —
   *but it can only do this for resources it can see directly in the markup*,
   not ones injected later by JavaScript.
3. **Resources are requested**, each with a browser-assigned priority
   (Highest / High / Medium / Low / Lowest). Images below the fold typically
   get a lower default priority than render-blocking CSS or fonts.
4. **Bytes arrive over the network**, are decoded (JPEG/PNG/WebP/AVIF
   decoding is CPU work, not just download time), and the image is painted.
5. **The browser continuously tracks the largest element painted so far.**
   Every time a bigger one appears, it becomes the new LCP candidate — until
   the page reports its **first user interaction** or is considered "settled,"
   at which point the final LCP value is locked in.

The reason a 9.6MB unoptimized JPEG is catastrophic isn't just "it's a big
file" — it's that step 3 and step 4 both get worse: the browser has to
transfer nearly ten megabytes *and* decode a 4200×2800 image before it can
paint anything, all before this element can even be considered "done."

---

## 3. How do you detect it? (Investigation Checklist)

Don't fix anything yet. Investigate first — this is the habit that matters.

**PageSpeed Insights / Lighthouse**
- [ ] Run Lighthouse (DevTools → Lighthouse tab → Performance → Analyze page load)
- [ ] Identify the reported LCP element (Lighthouse names it directly)
- [ ] Note the LCP time and the overall Performance score

**Network tab**
- [ ] Open DevTools → Network
- [ ] Check "Disable cache"
- [ ] Set throttling to "Fast 3G" or "Slow 4G"
- [ ] Reload the page
- [ ] Sort requests by **Size** — is `hero-image.jpg` the largest by far?
- [ ] Sort requests by **Time** — is it also the slowest?
- [ ] Click into the request and check the **Timing** tab (queuing, waiting, downloading)
- [ ] Check the **Preview** tab to confirm it's the right asset
- [ ] Check the **Headers** tab: response size, content-type, and cache headers
- [ ] Check the "Priority" column — what priority did the browser assign it?

**Performance panel**
- [ ] Open DevTools → Performance, record a reload
- [ ] Find the **LCP marker** on the timeline
- [ ] Confirm it lines up with the hero image finishing decode + paint

---

## 4. The intentionally broken code

See `index.html` — the bug is wrapped in
`<!-- INTENTIONAL PERFORMANCE ISSUE — LESSON 01 STARTS/ENDS HERE -->` comments.
Summary: a single `<img>` tag, no attributes beyond `src` and `alt`, pointing
at a 4200×2800, ~9.6MB JPEG saved at quality 95 with 4:4:4 chroma subsampling.

---

## 5. Why the browser behaves this way — "Why is THIS the LCP?"

Chrome selects an element as the LCP candidate when it is:

- **Above the fold** — visible without scrolling on first paint
- **The largest visible element** by rendered area, compared to every other
  candidate (text blocks, other images) on screen
- **An eligible element type** — images, `<video>` poster frames, background
  images with `background-image`, and block-level text nodes all qualify
- **Rendered within the viewport**, not just present in the DOM

In this page, the hero image is large, above the fold, and visually dominant
compared to the heading text next to it — so it wins every single time,
regardless of how long it takes to arrive. A slow LCP element doesn't stop
being the LCP element; it just makes LCP *worse*. Understanding this is the
key insight: **fixing LCP is about making the winning element fast, not about
trying to make something else win instead.**

---

## 6. Fix one thing at a time

Don't apply every fix at once — you won't know which change actually mattered.
Work through these versions in order, re-running Lighthouse after each one,
and fill in your own real numbers (the scores below are an illustrative
*typical* range, not a guarantee — your exact numbers depend on your device,
network conditions, and Lighthouse version).

| Version | Change | Typical score range |
|---|---|---|
| A | Original — unresized, quality 95, plain JPEG | ~10–25 |
| B | Resize to real display dimensions only | ~20–35 |
| C | + Re-compress with mozjpeg, quality ~80 | ~30–45 |
| D | + Convert to WebP | ~40–55 |
| E | + Convert to AVIF instead | ~50–65 |
| F | + Add `fetchpriority="high"` | ~55–70 |
| G | + Responsive `srcset`/`sizes` | ~65–80 |
| H | + `<link rel="preload">` for the LCP image | ~75–90 |
| I | + `width`/`height` attributes set correctly | ~80–95+ |

**Expected waterfall, before (Version A):**

```
Document        25 KB     ~80ms
CSS             20 KB     ~50ms
Hero Image     9.6 MB     ~15-20s  ← dominates everything
Everything else  ...      (queued behind the hero image on slow connections)
```

**Expected waterfall, after (Version I):**

```
Document        25 KB     ~80ms
CSS             20 KB     ~50ms
Hero AVIF     ~110 KB     ~200-400ms   ← fetched at high priority, in parallel
Everything else  ...      (no longer blocked behind a single giant request)
```

---

## 7. Common mistakes

- ❌ Preloading *every* image "just in case" — this competes with your actual
  LCP image for bandwidth and can make things worse
- ❌ Lazy-loading the hero/above-fold image — `loading="lazy"` on an LCP
  candidate directly delays the metric you're trying to fix
- ❌ Uploading a huge PNG for a photo — PNG is lossless and usually far
  larger than JPEG/WebP/AVIF for photographic content; use PNG for graphics
  with flat color or transparency, not photos
- ❌ Forgetting `srcset` — one giant image "works" on every device, but wastes
  bandwidth on every device that isn't the biggest one
- ❌ Ignoring `width`/`height` — even a perfectly compressed image can still
  contribute to layout shift without them
- ❌ Preloading the JPEG fallback while the browser actually downloads the
  AVIF — make sure your preload's `imagesrcset`/`type` matches what will
  really be selected, or you've just added a wasted download
- ❌ Slapping `fetchpriority="high"` on every image on the page — it only
  means something if it's relative to other resources; if everything is
  "high priority," nothing is

---

## 8. Real-world examples

You'll recognize this exact problem on:

- ✔ WordPress sites with an unoptimized theme hero/slider
- ✔ Elementor / page-builder sites where a client uploaded a phone photo directly
- ✔ Shopify / WooCommerce product hero banners
- ✔ Agency landing pages built for a pitch deck, never revisited for perf
- ✔ Hotel and restaurant sites (notoriously photo-heavy, rarely optimized)
- ✔ Corporate "About us" pages with a giant stock photo header
- ✔ React/Vue sites where a raw `<img src={import(...)}>` bypasses any
  build-time image optimization

---

## 9. Verification checklist

- [ ] Lighthouse LCP is in the "Good" range (≤2.5s) on a throttled connection
- [ ] "Properly size images" audit passes
- [ ] "Efficiently encode images" audit passes
- [ ] "Serve images in next-gen formats" audit passes
- [ ] "Improve image delivery" audit passes
- [ ] Network tab confirms the smallest applicable format/size was actually
      selected (check the request, not just the code)
- [ ] No new CLS introduced by your changes (check the CLS score didn't regress)

---

## 10. Summary & key takeaways

- LCP is decided by *which element is largest*, not by how fast it loads —
  slow doesn't disqualify it, it just makes the score worse
- File size, dimensions, format, and priority hints are four **separate**
  levers — most real pages need more than one of them
- Fix incrementally and re-measure after each change — the size of each
  score jump is the actual lesson, not just the final number
- Everything here — resize, compress, modern format, `srcset`, priority
  hints, preload, dimensions — generalizes directly to almost every
  image-heavy site you'll ever work on

---

## Track your own metrics

Fill this in as you go. This becomes your personal benchmark history.

| | Score | LCP | FCP | CLS | Speed Index | Transfer size |
|---|---|---|---|---|---|---|
| Before (Version A) | | | | | | |
| After your fix | | | | | | |

---

## Questions

Try to answer these without re-reading the guide:

1. Why was the plain JPEG version so much slower than everything else?
2. Why would adding `loading="lazy"` here make things *worse*, not better?
3. Would `preload` help if you added it to *every* image on the page?
4. What's the actual difference between `fetchpriority` and `preload` —
   aren't they doing the same thing?
5. Could adding `width`/`height` alone improve the *LCP* time itself, or
   only prevent layout shift?
6. What would happen to this page's LCP on a fast network but a very slow,
   low-end CPU?
