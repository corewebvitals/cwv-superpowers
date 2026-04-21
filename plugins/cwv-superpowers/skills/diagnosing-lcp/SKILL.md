---
name: diagnosing-lcp
description: Use when diagnosing LCP (Largest Contentful Paint) issues — slow hero images, slow page loads, poor Largest Contentful Paint at p75, or when the cwv-superpower orchestrator dispatches at Step 2B. Identifies the LCP element, the bottleneck phase (TTFB/LOADDELAY/LOADTIME/RENDERDELAY), and the cause pattern. Requires CoreDash MCP.
version: 2.0.2
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Diagnosing LCP

## Invocation

Called by the `cwv-superpower` orchestrator at Step 2B, or invoked directly when the user names LCP as the target metric.

**Inputs:** a target filter (`ff` or `u` with value) and a device (default: mobile).

Run the attribution, breakdown, and trend calls below, then interpret results.

**Progress:** Print these `↳` sub-status lines as you execute each call:
```
   ↳ Identifying LCP element...
   ↳ Checking element type...
   ↳ Checking priority state...
   ↳ Breaking down LCP phases...
   ↳ Analyzing distribution shape... (if warranted)
   ↳ Checking 7-day trend...
```

---

## Attribution calls

Use `[FILTER_KEY]` and `[FILTER_VALUE]` as placeholders — SKILL.md fills these based on the selected page filter.

### 1. LCP Element (CSS selector)

```
get_metrics with:
  metrics: "LCP"
  group: "lcpel"
  [FILTER_KEY]: "[FILTER_VALUE]"
  d: "mobile"
```

Identifies the LCP element as a CSS selector. The top result by traffic volume is the most common LCP element on the page. Use this to know exactly which DOM element is the LCP.

### 2. Element Type

```
get_metrics with:
  metrics: "LCP"
  group: "lcpet"
  [FILTER_KEY]: "[FILTER_VALUE]"
  d: "mobile"
```

Returns the type of the LCP element. Possible values: `image`, `text`, `background-image`, `video`. This determines which optimization strategies apply — image LCPs have different fixes than text LCPs.

### 3. Priority State

```
get_metrics with:
  metrics: "LCP"
  group: "lcpprio"
  [FILTER_KEY]: "[FILTER_VALUE]"
  d: "mobile"
```

Returns the priority/loading state of the LCP resource. Values:

| Value | Meaning |
|-------|---------|
| 0 | Not an image |
| 1 | Preloaded |
| 2 | High fetchpriority |
| 3 | Not preloaded |
| 4 | Lazy loaded |

**Key insight:** Priority 3 (not preloaded) is the most common fixable LCP issue — the image exists but the browser discovers it late. Priority 4 (lazy loaded) is actively harmful for LCP because `loading="lazy"` defers the load until the element is near the viewport, which directly delays the largest paint.

---

## Breakdown calls

### LCP Phase Breakdown

```
get_metrics with:
  metrics: "TTFB,LOADDELAY,LOADTIME,RENDERDELAY"
  [FILTER_KEY]: "[FILTER_VALUE]"
  d: "mobile"
```

Returns p75 values for each of the four LCP sub-phases.

**CRITICAL: Interpret proportionally.** Calculate each phase as a percentage of total LCP (sum of all four phases). The phase with the largest percentage share is the bottleneck. Do NOT judge phases by absolute thresholds — a 400ms LOADDELAY is fine when total LCP is 4000ms (10%) but terrible when total LCP is 800ms (50%).

**Example:**
- LCP = 3800ms total
- TTFB = 600ms (16%)
- LOADDELAY = 2100ms (55%)
- LOADTIME = 800ms (21%)
- RENDERDELAY = 300ms (8%)
- Bottleneck = **LOADDELAY** (55% of total LCP)

---

## Distribution investigation (optional)

If the breakdown findings raise a question — severity doesn't match the phases, the p75 is borderline (within 10% of the 2500ms threshold), or the attribution shows multiple LCP elements with different types — use `get_histogram` to check the distribution shape.

Print: `   ↳ Analyzing distribution shape...`

```
get_histogram with:
  metric: "LCP"
  filters: { "[FILTER_KEY]": "[FILTER_VALUE]", "d": "mobile" }
  date: "-7d"
```

Ask yourself: **"Does this distribution change the story?"** If the shape reveals something the p75 masked — a subpopulation getting hammered, a bimodal split between fast text-LCP users and slow image-LCP users, a heavy tail extending well past 4000ms — describe what you see and how it changes the diagnosis. If the shape confirms what you already know, move on without mentioning it.

If the shape suggests distinct user groups, you may call 1-2 filtered histograms to isolate what separates them (device, visitor type, network speed). Only do this if it would change the fix.

**Caveat:** Histograms can mislead. Mixed segments produce artificial shapes. Fixed 250ms bucket widths can split natural clusters. Cross-reference with the breakdown and attribution data.

If the diagnosis is already clear-cut from the breakdown, skip this section entirely.

---

## Trend call

```
get_timeseries with:
  metrics: "LCP"
  filters: { "[FILTER_KEY]": "[FILTER_VALUE]", "d": "mobile" }
  date: "-7d"
  granularity: "day"
```

Check the `summary.trend` field in the response to determine if LCP is improving, stable, or regressing over the past 7 days.

---

## Interpretation — what each bottleneck means

### TTFB bottleneck

**What it means:** The server is slow to deliver the HTML document. Everything else is waiting on the first byte.

**Common causes:**
- No CDN or CDN cache miss (HTML served from origin)
- Slow server-side rendering (SSR) — complex database queries, unoptimized templates
- Redirect chains (HTTP to HTTPS, www to non-www, vanity URLs)
- Geographic distance between user and server with no edge caching
- Slow DNS resolution or TLS handshake on cold connections

**Chrome trace goals:**
- Measure navigation start to first byte received in the Network panel
- Check for redirect entries in the timing breakdown (301/302 chains)
- Note the server response time (TTFB column in Network)
- Look at the "Waiting for server response" bar in the request timing waterfall
- Compare with and without cache to isolate server vs. network time

---

### LOADDELAY bottleneck

**What it means:** The LCP resource is discovered late by the browser. The HTML arrived, but the browser did not start fetching the LCP resource quickly enough.

**Common causes:**
- Image specified via CSS `background-image` (invisible to the preload scanner)
- Image injected via JavaScript (client-side rendering, lazy hydration frameworks)
- No `<link rel="preload">` hint for the LCP resource
- Render-blocking scripts in `<head>` delaying HTML parsing and resource discovery
- `loading="lazy"` on the LCP image causing deferred fetch
- Low fetch priority allowing other resources to queue ahead

**Chrome trace goals:**
- Find the LCP resource request in the Network waterfall
- Measure the gap between HTML first byte received and the LCP resource request start
- Look at what fills that gap: blocking scripts? CSS downloads? JS evaluation?
- Check whether the resource URL appears in the HTML source or is loaded via CSS/JS
- Verify if a preload hint exists in `<head>` for the LCP resource
- Check the `fetchpriority` attribute and `loading` attribute on the LCP element

---

### LOADTIME bottleneck

**What it means:** The LCP resource was discovered on time, but the download itself takes too long. The file is too large or the connection is too slow.

**Common causes:**
- Unoptimized image format — JPEG/PNG instead of WebP/AVIF
- No image CDN (images served from origin without optimization)
- No responsive `srcset` — full desktop-size image delivered to mobile devices
- Network contention from too many concurrent requests competing for bandwidth
- No HTTP/2 or HTTP/3 multiplexing, causing head-of-line blocking
- Missing or ineffective compression (no Brotli/gzip on text-based LCP)

**Chrome trace goals:**
- Find the LCP resource in the Network panel and note its download duration
- Check file size and format (Content-Type header, transferred vs. actual size)
- Look for concurrent request contention — are many resources downloading at the same time?
- Check CDN cache headers (`x-cache`, `cf-cache-status`, `age`) to confirm edge delivery
- Compare `Content-Length` against what a properly optimized version would be
- Check if the response uses HTTP/2+ (Protocol column in Network)

---

### RENDERDELAY bottleneck

**What it means:** The LCP resource finished loading, but the browser cannot paint it yet. Something is blocking the render.

**Common causes:**
- Render-blocking stylesheets still loading when the LCP resource is ready
- Synchronous `<script>` tags in `<head>` blocking the render pipeline
- Large JavaScript bundle evaluation on the main thread delaying layout/paint
- A/B testing or personalization scripts delaying element visibility
- `display: none` or `visibility: hidden` toggled by JS after load
- Font loading delays when LCP is a text element (FOIT)

**Chrome trace goals:**
- Find the gap between LCP resource download complete and the LCP paint event in the Performance panel
- Look for long tasks on the main thread during that gap (yellow blocks > 50ms)
- Check if render-blocking CSS or scripts are still loading after the LCP resource finishes
- Look for JS-driven visibility changes (`display`, `visibility`, `opacity`) on the LCP element
- Check the "Rendering" tab for forced style recalculations or layout thrashing
- For text LCP: check font loading in Network and whether `font-display: swap` or `optional` is used

---

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| TTFB is only 700ms so it's not the bottleneck | Proportional share matters, not absolute. 700ms at 58% of 1200ms LCP IS the bottleneck. Report the dominant phase regardless of whether its absolute value is "good." |
| `lcpel` selector didn't match — skip attribution | Fall back to `lcpet` (element type) and `lcpprio` (fetch priority). Don't give up. Selectors break on minified markup; type + priority still identify the element class. |
| Chrome-measured LCP doesn't match RUM LCP | Expected. Chrome shows structure, not user timing. Don't recalibrate RUM findings to Chrome — RUM is truth for the p75 you're fixing. |
| LCP p75 is 2400ms so it passes Core Web Vitals — move on | Check the tail. A passing p75 with 18% poor loads is failing for one user in six. Pull `get_histogram` and report the poor-bucket share. |
| Priority 3 (not preloaded) isn't a real issue | Priority 3 on the LCP image is the single most common fixable LCP issue. `fetchpriority="high"` or a `<link rel="preload">` typically removes 200–600ms of LOADDELAY. |
| Largest phase is only 45% — not really a bottleneck | 45% of LCP is still the phase to fix first. Proportional share is the tiebreaker; do not require >50%. |

## Red Flags

- Explaining LCP in absolute thresholds ("700ms is fine") instead of phase percentages.
- Recommending "optimize the image" without identifying which phase (LOADDELAY vs LOADTIME) dominates.
- Concluding "can't fix TTFB" without checking redirect chains, server timing headers, or CDN edge placement.
- Skipping the histogram check because the p75 passes.
- Naming a generic fix ("preload your LCP") without confirming the element, its current priority, and its discovery path.

---

## Summary format

After running all calls, print `✅ LCP diagnosed: [BOTTLENECK PHASE] is [X]% of LCP — [one-sentence cause]`

Then produce this structured summary for SKILL.md:

```
LCP diagnosis for [PAGE]:
- p75: [VALUE]ms ([RATING]) | Mobile: [VALUE]ms | Desktop: [VALUE]ms
- Distribution: [X]% good / [Y]% needs improvement / [Z]% poor
- LCP element: [CSS SELECTOR] ([TYPE])
- Priority state: [0-4] — [EXPLANATION]
- Bottleneck phase: [PHASE] ([X]% of total LCP)
  - TTFB: [VALUE]ms ([X]%)
  - LOADDELAY: [VALUE]ms ([X]%)
  - LOADTIME: [VALUE]ms ([X]%)
  - RENDERDELAY: [VALUE]ms ([X]%)
- Trend (7d): [improving/stable/regressing] ([X]% change)
- Distribution shape: [observation — what it means for this diagnosis] (only if histogram was used)
- Chrome should investigate: [SPECIFIC TRACE GOAL FROM BOTTLENECK SECTION]
```
