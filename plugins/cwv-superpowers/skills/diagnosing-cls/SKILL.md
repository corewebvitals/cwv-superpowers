---
name: diagnosing-cls
description: Use when diagnosing CLS (Cumulative Layout Shift) issues — visual layout jumps, content shifting during load, poor CLS at p75, or when the cwv-superpower orchestrator dispatches at Step 2B. Matches the shifting element to one of 5 cause patterns (images without dimensions, FOUT, injected content, late-loading resources, non-composited animations). Requires CoreDash MCP.
version: 2.0.1
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Diagnosing CLS

CLS has no phase breakdown — `BREAKDOWN.CLS` is `["CLS"]`. Diagnosis works by identifying the shifting element and matching it to a cause pattern through cross-dimensional analysis (device type, visitor type, network speed).

## Invocation

Called by the `cwv-superpower` orchestrator at Step 2B, or invoked directly when the user names CLS as the target metric.

**Inputs:** a target filter (`ff` or `u` with value) and a device (default: mobile).

**Progress:** Print these `↳` sub-status lines as you execute each call:
```
   ↳ Identifying shifting element...
   ↳ Comparing mobile vs desktop...
   ↳ Comparing new vs repeat visitors...
   ↳ Checking network speed sensitivity...
   ↳ Checking 7-day trend...
   ↳ Analyzing distribution shape... (if warranted)
```

---

## Attribution calls

All calls use the page filter `[FILTER_KEY]: "[FILTER_VALUE]"` established by the caller.

### 1. Shifting element identification

```
get_metrics metric: CLS, group: clsel, [FILTER_KEY]: "[FILTER_VALUE]"
```

Returns CLS contribution per CSS selector. The top result is the biggest CLS contributor — this is the shifting element to diagnose.

### 2. Mobile vs desktop split

```
get_metrics metric: CLS, [FILTER_KEY]: "[FILTER_VALUE]", d: "mobile"
get_metrics metric: CLS, [FILTER_KEY]: "[FILTER_VALUE]", d: "desktop"
```

Mobile-only CLS causes:
- Images without responsive sizing (no `width`/`height` attributes or CSS `aspect-ratio`)
- Ads without reserved space collapsing or expanding on small viewports
- Font metric differences more pronounced on small screens

Desktop-only CLS (rare):
- Sidebar widgets loading late and pushing main content
- Sticky headers recalculating on wider viewports

### 3. New vs repeat visitor split

```
get_metrics metric: CLS, [FILTER_KEY]: "[FILTER_VALUE]", fv: "1"
get_metrics metric: CLS, [FILTER_KEY]: "[FILTER_VALUE]", fv: "0"
```

`fv` values: `0` = repeat visitor, `1` = new visitor, `2` = not measured (via settings).

A significant gap where new visitors have worse CLS points to uncached resources causing shifts (web fonts, third-party scripts loaded fresh).

### 4. Network speed breakdown

```
get_metrics metric: CLS, group: dl, [FILTER_KEY]: "[FILTER_VALUE]"
```

`dl` = network speed in Mbps. If CLS worsens at lower `dl` values, the shift is caused by slow-loading resources that arrive after initial paint.

---

## Trend call

```
get_timeseries metric: CLS, filters: { "[FILTER_KEY]": "[FILTER_VALUE]" }, date: "-7d", granularity: "day"
```

Interpretation:
- A sudden spike = a deploy introduced the shift (check release timing).
- Steady elevated CLS = structural issue baked into the page template.

---

## Distribution investigation (optional)

If the cross-dimensional analysis leaves the cause unclear, or CLS is borderline (within 20% of the 0.1 threshold), use `get_histogram` to check the distribution shape.

Print: `   ↳ Analyzing distribution shape...`

```
get_histogram with:
  metric: "CLS"
  filters: { "[FILTER_KEY]": "[FILTER_VALUE]" }
  date: "-7d"
```

Ask yourself: **"Does this distribution change the story?"** CLS distributions are often heavily left-skewed — most users score 0 or near-0, and a smaller group gets hit hard. The histogram reveals how concentrated the damage is: a few users with extreme shifts (likely a specific viewport or interaction trigger) versus a broad population with moderate shifts (likely a structural template issue).

If the shape suggests distinct groups, you may call 1-2 filtered histograms to isolate what separates them (device type is the most common split for CLS). Only do this if it would change the fix.

**Caveat:** Histograms can mislead. Mixed segments produce artificial shapes. Fixed 0.025 bucket widths can split natural clusters. Cross-reference with the device, visitor type, and network speed analysis.

If the cause pattern is already clear from the cross-dimensional analysis, skip this section entirely.

---

## Cause pattern matching

Walk through these patterns in order. Stop at the first match. Each pattern defines: signal, why it shifts, Chrome trace goals, and fix pattern.

### Pattern 1: Image or video without dimensions

**Signal:** The shifting element (from `clsel`) is an `img`, `video`, or a container element wrapping one of these.

**Why it shifts:** The browser cannot reserve space for the media before the resource loads. Once the image or video arrives, content below it pushes down by the full height of the media element.

**Chrome trace goals:** Find the element in the DOM. Check whether it has `width` and `height` HTML attributes. Check for a CSS `aspect-ratio` rule. Take a screenshot before the image loads and after to confirm the layout shift.

**Fix pattern:** Add explicit `width` and `height` attributes to the element. For responsive layouts, combine with CSS `width: 100%; height: auto;` — the browser calculates the aspect ratio from the HTML attributes and reserves the correct space at any viewport width.

---

### Pattern 2: Web font swap (FOUT)

**Signal:** CLS is significantly worse for new visitors (`fv: "1"`) than repeat visitors (`fv: "0"`). The shifting element is a text block (`p`, `h1`, `h2`, `span`, `div` containing text, etc.).

**Why it shifts:** The fallback font renders first, then the web font loads and the text reflows because the font metrics (x-height, character widths, line height) differ between fallback and web font. Repeat visitors have the font cached, so no swap occurs.

**Chrome trace goals:** Find font loading in the network waterfall. Capture the exact moment text reflows from fallback to web font. Compare fallback font metrics against web font metrics to measure the size difference.

**Fix pattern:** Use `font-display: optional` to eliminate FOUT entirely — the web font is only used if it is already cached, otherwise the fallback renders with no swap. Alternatively, use `font-display: swap` combined with `size-adjust`, `ascent-override`, and `descent-override` on the `@font-face` rule to match the fallback font metrics to the web font. Preload critical fonts with `<link rel="preload" as="font" crossorigin>`.

---

### Pattern 3: Dynamically injected content

**Signal:** The shifting element is a container (`div`, `section`, `aside`) near the top of the page. CLS is not dependent on network speed. CLS is the same for new and repeat visitors.

**Why it shifts:** Content such as an ad slot, cookie consent banner, notification bar, or personalization widget is injected into the DOM after initial render. The insertion pushes all content below it down.

**Chrome trace goals:** Find the DOM insertion event in the Performance timeline. Identify the script responsible for injecting the content. Take a screenshot before and after the injection to confirm the layout shift.

**Fix pattern:** Reserve fixed space for the injected content using `min-height` on the container element. Alternatively, render the injected content as an overlay or `position: fixed` element instead of inserting it inline in the document flow.

---

### Pattern 4: Late-loading resource pushing content

**Signal:** CLS is worse on slow connections (low `dl` values, e.g., 1.5 Mbps or below).

**Why it shifts:** On slow connections, images and embeds above the fold load later than on fast connections. When these resources finally render, they push surrounding content and cause layout shifts.

**Chrome trace goals:** Find resources that complete loading after initial paint. Trace the cascade of elements that shifted as a result of each late-arriving resource.

**Fix pattern:** Add explicit dimensions (`width` and `height` attributes) to all above-fold images. Use CSS `aspect-ratio` on embed containers (iframes, video embeds). Apply `loading="lazy"` to below-fold content that is shifting when it scrolls into view.

---

### Pattern 5: CSS animation using layout properties

**Signal:** CLS is consistent across all segments — same on mobile and desktop, same for new and repeat visitors, no correlation with network speed.

**Why it shifts:** Animating layout-triggering CSS properties (`top`, `left`, `width`, `height`, `margin`, `padding`) forces the browser to recalculate layout on every animation frame. Each recalculation contributes to the cumulative layout shift score.

**Chrome trace goals:** Check the Animations panel for non-composited animations. Look for repeated layout recalculations occurring during animation frames in the Performance timeline.

**Fix pattern:** Replace layout-triggering animations with `transform` (`translate` for position, `scale` for size) and `opacity`. These properties are composited on the GPU and do not cause layout shifts.

---

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| CLS is 0.11 — basically fine | The threshold is 0.1. 0.11 at p75 is poor. The histogram tail rule applies to CLS too — check the poor-bucket share before moving on. |
| Can't find `clsel` element in the DOM | Selectors may be minified or generated. Fuzzy-match by position, role, or type. Two-step matching (exact → fallback) is mandatory; don't stop at step one. |
| `fv` (new vs repeat) gap is only 15% — not a font/cache issue | Only skip Pattern 2 (FOUT) if the gap is under ~10%. 15% is still enough evidence to investigate font loading. |
| Animations use `transform` so they can't cause shift | Check elsewhere for `width`, `height`, `margin`, `padding`, or `top/left` animations. One handler per pattern — walk all five before concluding. |
| Only a few pages are affected — low priority | CLS compounds across a session; one shifty page taints the site-wide p75. Use `ff` to find the offending template, not the count. |
| CLS is hard to reproduce in Chrome | Expected. CLS depends on network timing, font latency, and third-party ad loads that Chrome's lab conditions don't replicate. Trust RUM `clsel` data over local repro. |

## Red Flags

- Concluding "it's a structural template issue" without walking the 5 patterns in order.
- Stopping at the first signal that matches weakly instead of checking all five.
- Recommending "set image dimensions" without confirming the shifting element is an image.
- Ignoring the `fv` split because the gap "looks small."
- Reporting CLS as a single number without naming the pattern and the element.

---

## Summary format

After running all calls, print `✅ CLS diagnosed: [PATTERN NAME] — [one-sentence cause]`

```
CLS diagnosis for [PAGE]:
- p75: [SCORE] ([RATING]) | Mobile: [SCORE] | Desktop: [SCORE]
- Distribution: [X]% good / [Y]% needs improvement / [Z]% poor
- Shifting element: [CSS SELECTOR]
- New visitor CLS: [SCORE] | Repeat visitor CLS: [SCORE] | Difference: [significant/minimal]
- Network sensitivity: [worse on slow connections: yes/no]
- Likely cause: [PATTERN NAME] — [ONE SENTENCE EXPLANATION]
- Trend (7d): [improving/stable/regressing] ([X]% change)
- Distribution shape: [observation — what it means for this diagnosis] (only if histogram was used)
- Chrome should investigate: [SPECIFIC TRACE GOAL FROM MATCHED PATTERN]
```
