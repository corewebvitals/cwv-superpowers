---
name: tracing-with-chrome
description: Use when running a Chrome lab trace for a specific CWV bottleneck — investigating LCP/INP/CLS with browser tools after RUM has identified the target phase, element, and cause. Typically called by the cwv-superpower orchestrator at Step 3. Requires Chrome via chrome-devtools MCP.
version: 2.0.1
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Tracing with Chrome

## Invocation

Called by the `cwv-superpower` orchestrator at Step 3, or invoked directly when the user has a specific URL, metric, and suspected bottleneck to investigate.

**Inputs:**
- **Target URL** — the page to trace.
- **Metric** — LCP, INP, or CLS.
- **Bottleneck phase or cause pattern** — from the diagnosis skill (e.g., "LOADDELAY", "INPUTDELAY", "font swap on new visitors"). Drives what to look for.
- **Element selector** — from CoreDash attribution (if available).

Chrome investigates the specific bottleneck phase identified by RUM. Not all phases — just the one that matters. The trace also collects visual evidence for the report: network waterfall, page filmstrip, INP interaction timeline.

---

## Core Rules
Three non-negotiable rules:

1. **RUM found the issue — it IS there.** If Chrome shows fast timing on the dev machine, that's expected. The dev machine is faster than a median mobile user on a mid-range Android on a 3G connection. The trace shows the SEQUENCE and STRUCTURE of the problem, not the user's timing. Never conclude "the issue doesn't exist" based on lab results.

2. **Never say "I couldn't reproduce the issue."** The issue is confirmed by hundreds or thousands of real user sessions. Chrome is here to show WHY it happens, not IF it happens.

3. **Fuzzy element matching — two steps:**
   - Step 1: Try the exact CSS selector from CoreDash in the DOM.
   - Step 2: If not found, find the most likely element that matches what the selector describes — by position, role, and type. A selector like `section.hero > img.css-x7k2m3` means "the image inside the hero section." Find that image regardless of class names. Class names may be generated, minified, or changed between builds.

## Progress

Print these `↳` sub-status lines as you execute each action:
```
   ↳ Navigating to page...
   ↳ Capturing performance trace...
   ↳ Analyzing [waterfall/flamechart/filmstrip]...
```

## Trace Setup
- **Mobile analysis (default):** Device emulation (mobile viewport, e.g. Moto G Power), Network: Fast 3G throttling, CPU: 4x slowdown. These approximate a mid-range mobile device on a typical mobile network.
- **Desktop analysis:** No throttling, no CPU slowdown, standard viewport.
- **Recording:** Start recording before navigation. Navigate to target URL. For LCP/CLS: wait for page load to complete (load event + 2 seconds for late shifts). For INP: after page load, trigger the specific interaction identified by `inpel` attribution.

## LCP Trace Investigation

Guided by the bottleneck phase from the diagnosis module. Focus on the bottleneck — don't investigate all phases.

### If bottleneck is TTFB
1. In the Network panel, find the initial HTML document request
2. Measure time from navigation start to first byte received (the "Waiting for server response" bar)
3. Check for redirect entries in the timing breakdown — each 301/302 hop adds a full round trip
4. Note the server response time from the request timing waterfall
5. Compare: is the delay in DNS, connection, TLS, or server processing?

### If bottleneck is LOADDELAY
1. Find the LCP resource (image, font, or video) in the Network waterfall
2. Measure the gap between HTML first byte arrival and when the LCP resource request starts — this gap IS the load delay
3. Look at what fills that gap:
   - Are there render-blocking `<script>` tags in `<head>` delaying HTML parsing?
   - Is the resource referenced in CSS or loaded via JavaScript instead of directly in HTML `<img>` tags?
   - Are other lower-priority resources being fetched first?
4. Check the HTML source: is there a `<link rel="preload">` for this resource?
5. Check the LCP element: does it have `fetchpriority="high"`? Does it have `loading="lazy"` (harmful)?
6. If the resource is a CSS `background-image`, the preload scanner cannot discover it — this is the root cause

### If bottleneck is LOADTIME
1. Find the LCP resource in the Network panel — click it to see the timing breakdown
2. Note the download duration (Content Download time), not just total time
3. Check response headers:
   - File size (`Content-Length`) and transferred size (with compression)
   - Format (`Content-Type` — JPEG/PNG vs WebP/AVIF?)
   - CDN cache status (`x-cache`, `cf-cache-status`, `x-vercel-cache`, `age`)
   - Compression (`Content-Encoding: br` or `gzip`)
4. Look for network contention: are many other resources downloading simultaneously, competing for bandwidth?
5. Check the Protocol column — is the connection using HTTP/2 or HTTP/3?

### If bottleneck is RENDERDELAY
1. In the Performance panel, find the LCP paint event (marked in the Timings row)
2. Find when the LCP resource finished downloading in the Network section
3. Measure the gap between resource complete and LCP paint — this gap IS the render delay
4. In the flamechart, look for long tasks (yellow blocks > 50ms) between resource completion and paint:
   - Script evaluation of large JS bundles?
   - Style recalculation from large CSS files?
   - JavaScript toggling element visibility (`display`, `opacity`)?
5. Check if render-blocking stylesheets or scripts are still loading after the LCP resource finishes
6. For text LCP: check if font loading is causing the delay (FOIT)

## INP Trace Investigation

1. Wait for page to fully load (load event fires)
2. Start recording in the Performance panel (or mark the timeline)
3. Trigger the exact interaction identified by `inpel` — click the button, type in the input, tap the link
4. Stop recording after the visual update completes

Then investigate based on the bottleneck phase:

### If bottleneck is INPUTDELAY
1. In the flamechart, find the interaction event marker (the click/keypress event)
2. Look at main thread activity IMMEDIATELY BEFORE the event handler starts executing
3. Identify the long task(s) blocking the thread:
   - Which script is running? Match the filename against the `lurl` from CoreDash
   - Is it a timer callback (`setTimeout`/`setInterval`)? Script evaluation? Framework hydration?
4. Measure the time from the interaction marker to when the event handler actually starts
5. If the diagnosis module reported load state `loading`/`dom-interactive`, look specifically for script evaluation tasks (the page was still booting up when the user interacted)

### If bottleneck is PROCESSING
1. Find the event handler function(s) in the flamechart — they run immediately after the interaction event
2. Measure total handler execution time
3. Look inside the handler for:
   - Purple "Layout" bars = forced reflow / layout thrashing (DOM read after DOM write)
   - Long JavaScript execution blocks (heavy computation, large array operations)
   - Synchronous API calls or large `localStorage` operations
   - Framework re-render cycles (React reconciliation, Vue patching) — note how many components re-rendered
4. Identify the single function call that takes the most time within the handler

### If bottleneck is PRESENTATION
1. Look at rendering activity AFTER the event handler completes
2. Find "Recalculate Style", "Layout", and "Paint" blocks — measure their combined duration
3. Check DOM size: run `document.querySelectorAll('*').length` in the Console — values >1,500 are a concern
4. Look for non-composited animations triggered by the interaction (check Animations panel)
5. Check if `content-visibility: auto` on off-screen sections could reduce the layout scope

## CLS Trace Investigation

In the Performance panel, look for red "Layout Shift" markers in the Experience row. Click each to see which element moved and the shift score. Then investigate based on the cause pattern from the diagnosis module:

### For image/video without dimensions
1. Find the element in the DOM (Elements panel)
2. Confirm it has no `width` and `height` HTML attributes
3. Check for CSS `aspect-ratio` on the element or its container
4. Screenshot before the resource loads (space collapsed) and after (space expanded)

### For font swap (FOUT)
1. Find the web font file(s) in the Network waterfall
2. Note when the font finishes loading relative to First Contentful Paint
3. Watch for the text reflow moment — screenshot before (fallback font) and after (web font)
4. Compare the visual difference to gauge the metric mismatch between fonts

### For injected content
1. In the Performance timeline, find the DOM insertion event that causes the shift
2. Identify which script injected the content (trace the call stack)
3. Note the timing: how long after initial paint does the injection happen?
4. Screenshot before and after the injection to confirm what shifted

### For late-loading resource
1. Find the late-loading resource in the Network waterfall
2. Note its completion time relative to First Contentful Paint
3. Trace the cascade: when it loads, which elements shift and by how much?
4. Note whether the shift happens above or below the fold

### For CSS animation
1. Open the Animations panel and look for non-composited animations
2. In the Performance panel, look for repeated "Layout" recalculations during animation frames
3. Identify which CSS properties are being animated (look for `top`, `left`, `width`, `height`, `margin`, `padding`)

## Evidence Capture

Collect data and screenshots using Playwright MCP tools. These feed into the report template's SVG renderers and filmstrip section.

### Network waterfall data (for SVG waterfall in report)
1. Run `browser_network_requests` to get all HTTP requests with timing.
2. Run `browser_evaluate` with this script to get per-resource timing breakdown:
   ```js
   JSON.stringify(performance.getEntriesByType('resource').map(e => ({
     name: e.name.split('/').pop().split('?')[0] || e.name,
     start: Math.round(e.startTime),
     dns: Math.round(e.domainLookupEnd - e.domainLookupStart),
     connect: Math.round(e.connectEnd - e.connectStart),
     tls: Math.round(e.secureConnectionStart > 0 ? e.connectEnd - e.secureConnectionStart : 0),
     ttfb: Math.round(e.responseStart - e.requestStart),
     download: Math.round(e.responseEnd - e.responseStart),
     size: e.transferSize,
     isLcp: false
   })))
   ```
3. Also get navigation timing for the document itself:
   ```js
   JSON.stringify((function(n) { return {
     name: 'document', start: 0,
     dns: Math.round(n.domainLookupEnd - n.domainLookupStart),
     connect: Math.round(n.connectEnd - n.connectStart),
     tls: Math.round(n.secureConnectionStart > 0 ? n.connectEnd - n.secureConnectionStart : 0),
     ttfb: Math.round(n.responseStart - n.requestStart),
     download: Math.round(n.responseEnd - n.responseStart),
     size: 0, isLcp: false
   }; })(performance.getEntriesByType('navigation')[0]))
   ```
4. Get the LCP time and resource URL:
   ```js
   JSON.stringify(await new Promise(r => new PerformanceObserver(l => {
     var entries = l.getEntries(); var last = entries[entries.length - 1];
     r({ time: Math.round(last.startTime), url: last.url || '', element: last.element?.tagName });
   }).observe({ type: 'largest-contentful-paint', buffered: true })))
   ```
5. When populating the report template, replace the `waterfallData` array and `lcpTime` value inside the `WATERFALL_DATA_START` / `WATERFALL_DATA_END` markers with real data. Set `isLcp: true` on the entry matching the LCP resource URL. Keep only the top 15-20 resources by start time to avoid visual clutter.

### Page filmstrip (real page screenshots)
1. Use `browser_navigate` to go to the target URL.
2. Take `browser_take_screenshot` immediately — this captures the blank/loading state. Label: "0.0s".
3. Run `browser_evaluate` to wait for first contentful paint:
   ```js
   await new Promise(r => new PerformanceObserver(l => r()).observe({ type: 'paint', buffered: true }));
   Math.round(performance.getEntriesByType('paint').find(e => e.name === 'first-contentful-paint')?.startTime || 0)
   ```
   Then take `browser_take_screenshot`. Label with the FCP time.
4. Run `browser_evaluate` to wait for LCP:
   ```js
   await new Promise(r => setTimeout(r, 3000));
   Math.round((await new Promise(r => new PerformanceObserver(l => {
     var e = l.getEntries(); r(e[e.length-1].startTime);
   }).observe({ type: 'largest-contentful-paint', buffered: true }))))
   ```
   Then take `browser_take_screenshot`. Label with the LCP time.
5. Wait 2 more seconds, take a final `browser_take_screenshot`. Label: "Loaded".
6. Encode each screenshot as a base64 data URI and insert into the filmstrip frames in the report template.

If timed screenshots are not possible (e.g. page loads too fast to capture intermediate states), capture at minimum ONE fully-loaded screenshot and insert it as the single filmstrip frame.

### Interaction timeline data (INP only)
1. Before triggering the interaction, run `browser_evaluate` to set up observers:
   ```js
   window.__cwvEvents = []; window.__cwvTasks = [];
   new PerformanceObserver(l => l.getEntries().forEach(e => window.__cwvEvents.push({
     name: e.name, start: Math.round(e.startTime),
     inputDelay: Math.round(e.processingStart - e.startTime),
     processing: Math.round(e.processingEnd - e.processingStart),
     presentation: Math.round(e.startTime + e.duration - e.processingEnd),
     duration: Math.round(e.duration)
   }))).observe({ type: 'event', buffered: true, durationThreshold: 16 });
   new PerformanceObserver(l => l.getEntries().forEach(e => window.__cwvTasks.push({
     start: Math.round(e.startTime), duration: Math.round(e.duration)
   }))).observe({ type: 'longtask', buffered: true });
   ```
2. Trigger the interaction (click, type, etc.).
3. Wait 1 second, then collect:
   ```js
   JSON.stringify({ events: window.__cwvEvents, longTasks: window.__cwvTasks })
   ```
4. Use this data in the report's INP timeline section to show the input delay / processing / presentation breakdown visually.

### Element screenshot
Use `browser_take_screenshot` of the full page (or a specific element if supported) with the identified problem element visible in the viewport. This goes into the report as supporting evidence.

## Output format

After completing the trace, print `✅ Confirmed: [one-sentence key finding from trace]`

```
Chrome trace findings:
- URL visited: [URL]
- Emulation: [mobile Fast 3G 4x CPU / desktop no throttling]
- Bottleneck phase confirmed: [PHASE] — [what Chrome showed]
- Root cause evidence: [2-3 sentences describing what was seen]
- Key measurement: [specific timing or gap that proves the bottleneck]
- Evidence captured: [filmstrip screenshots: N frames, waterfall data: N resources, element screenshot]
```
