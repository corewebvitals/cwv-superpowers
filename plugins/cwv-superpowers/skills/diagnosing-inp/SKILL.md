---
name: diagnosing-inp
description: Use when diagnosing INP (Interaction to Next Paint) issues — slow/laggy clicks, delayed interactions, poor INP at p75, or when the cwv-superpower orchestrator dispatches at Step 2B. Identifies the slow interaction element, the bottleneck phase (INPUTDELAY/PROCESSING/PRESENTATION), and the responsible LOAF script. Requires CoreDash MCP.
version: 2.0.1
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Diagnosing INP

## Invocation

Called by the `cwv-superpower` orchestrator at Step 2B, or invoked directly when the user names INP as the target metric.

**Inputs:** a target filter (`ff` or `u` with value) and a device (default: mobile).

**Progress:** Print these `↳` sub-status lines as you execute each call:
```
   ↳ Identifying slow interaction element...
   ↳ Finding responsible scripts (LOAF)...
   ↳ Checking interaction load state...
   ↳ Breaking down INP phases...
   ↳ Analyzing distribution shape... (if warranted)
   ↳ Checking 7-day trend...
```

---

## Attribution calls

All calls use the provided `[FILTER_KEY]`/`[FILTER_VALUE]` placeholders for page filtering and `d=mobile` (or specified device).

1. **`get_metrics`** with `group: "inpel"` filtered by `[FILTER_KEY]=[FILTER_VALUE]`, device.
   Returns the slow interaction element (CSS selector).
   Note all segments — sometimes multiple elements have poor INP (e.g. a carousel button AND a hamburger menu). Each may have a different root cause.

2. **`get_metrics`** with `group: "lurl"` filtered by `[FILTER_KEY]=[FILTER_VALUE]`, device.
   Returns LOAF script URLs — the scripts running Long Animation Frames during slow interactions.
   For each URL, determine: first-party or third-party?
   Cross-reference with `inpel` results — e.g. `carousel.bundle.js` + `button.carousel-next` = near-certain match between script and element.

3. **`get_metrics`** with `group: "inpls"` filtered by `[FILTER_KEY]=[FILTER_VALUE]`, device.
   Returns the load state when the interaction happened.
   Possible values: `loading`, `dom-interactive`, `dom-content-loaded`, `complete`.
   - If `loading` or `dom-interactive`: script loading order problem — the page was still parsing when users interacted. Users click/tap before the page is ready.
   - If `complete`: event handler code problem — the page was fully loaded, so the slowness lives in the handler logic itself.

## Breakdown calls

One call: **`get_metrics`** with `metrics: "INPUTDELAY,PROCESSING,PRESENTATION"` filtered by `[FILTER_KEY]=[FILTER_VALUE]`, device.

INP breakdown phases and thresholds (from cwvmetrics.js):
- **INPUTDELAY** — good: <40ms, needs improvement: 40-80ms, poor: >80ms
- **PROCESSING** — good: <80ms, needs improvement: 80-160ms, poor: >160ms
- **PRESENTATION** — good: <80ms, needs improvement: 80-160ms, poor: >160ms

**CRITICAL:** Use proportional interpretation — the phase with the largest percentage of total INP is the bottleneck.
Example: INP=310ms, INPUTDELAY=190ms (61%), PROCESSING=80ms (26%), PRESENTATION=40ms (13%) -> bottleneck = INPUTDELAY.
Do NOT rely solely on absolute thresholds. A phase can be within its "good" threshold but still be the dominant contributor if the others are even smaller.

---

## Distribution investigation (optional)

If the breakdown findings raise a question — the p75 is borderline (within 20% of the 200ms threshold), the `inpel` attribution shows multiple slow elements, or the load state suggests mixed populations — use `get_histogram` to check the distribution shape.

Print: `   ↳ Analyzing distribution shape...`

```
get_histogram with:
  metric: "INP"
  filters: { "[FILTER_KEY]": "[FILTER_VALUE]", "d": "mobile" }
  date: "-7d"
```

Ask yourself: **"Does this distribution change the story?"** INP distributions often have heavy tails — most interactions are fine but a specific interaction path is slow. If the histogram shows a long right tail, cross-reference with the `inpel` segments to identify which element/interaction is responsible for the tail. If the shape confirms a uniform problem, move on.

If the shape suggests distinct groups, you may call 1-2 filtered histograms to isolate what separates them. Only do this if it would change the fix.

**Caveat:** Histograms can mislead. Mixed segments produce artificial shapes. Fixed 25ms bucket widths can split natural clusters. Cross-reference with the breakdown, `inpel`, and `lurl` data.

If the diagnosis is already clear-cut from the breakdown and LOAF attribution, skip this section entirely.

---

## Trend call

**`get_timeseries`** — 7-day window, day granularity, for INP filtered by `[FILTER_KEY]=[FILTER_VALUE]`, device.

## Interpretation — what each bottleneck means

### INPUTDELAY bottleneck

Main thread was busy when the user interacted — the event handler couldn't start.

**Common causes:**
- Large JS bundles being evaluated at page load
- Third-party scripts (analytics, chat widgets, A/B testing tools) running scheduled tasks
- Timer functions (`setTimeout`/`setInterval`) firing at the wrong moment
- Overlapping interactions queued sequentially
- Framework hydration blocking the main thread (React, Next.js, Nuxt)

**Chrome trace goals:**
Trigger the interaction from `inpel`. In the flamechart, look at activity BEFORE the event handler starts. Identify the blocking long task(s) — which script is responsible? Correlate with `lurl` results from CoreDash. If load state was `loading` or `dom-interactive`, look for script evaluation tasks specifically.

### PROCESSING bottleneck

The event handler itself is doing too much work.

**Common causes:**
- Synchronous DOM reads after writes (layout thrashing / forced reflows)
- Heavy computation in the handler (filtering large lists, complex state updates)
- Synchronous `localStorage` reads
- Large framework re-renders (React reconciling thousands of components)
- Deep cloning or serialization of large objects

**Chrome trace goals:**
Find the event handler in the flamechart, measure its duration. Look for purple "Layout" bars inside the handler (forced reflow). Look for long JS execution blocks. If a framework re-render is visible, note the scope (how many components) and duration.

### PRESENTATION bottleneck

The handler finished but the browser was slow to paint the result.

**Common causes:**
- Very large DOM (>1500 nodes) requiring extensive style recalculation and layout
- Complex CSS selectors being recalculated across the tree
- Non-composited animations triggered by the interaction
- Large DOM insertions or removals causing full layout
- Missing `content-visibility: auto` on off-screen sections

**Chrome trace goals:**
Look at rendering activity after the handler completes. Find "Recalculate Style", "Layout", and "Paint" blocks — measure their duration. Check DOM size (`document.querySelectorAll('*').length`). Look for non-composited animations. Check if `content-visibility: auto` could reduce layout scope.

---

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| INP is 210ms — barely poor, skip it | 210ms at p75 is confirmed poor, not borderline. That's one in four interactions failing the threshold. Report it with the same urgency as 400ms. |
| No LOAF data returned — can't blame a specific script | Long Animation Frames may not have fired; cross-reference the `inpel` selector with the page's event handlers and third-party scripts. Absence of LOAF is recoverable, not a dead end. |
| Handler code is short, so it can't be processing-bound | Layout thrash inside a 20ms handler (reading `offsetHeight` in a loop, for example) can cost 200ms of forced reflow. Look for purple Layout bars in the trace, not just yellow Scripting. |
| INP is only slow for users who interact early (`inpls: loading`) | That's a script-order bug, not a population bug. Move the blocking script, defer it, or attach the handler after load. Don't dismiss it as "some users." |
| The breakdown shows PRESENTATION dominant — there's nothing to fix | PRESENTATION = rendering + paint after the handler returns. Non-composited animations, oversized DOM, missing `content-visibility` — all fixable. Keep digging. |
| Third-party script so it's not our problem | Cross-origin scripts still run in your main thread. Async-loading, `requestIdleCallback` wrapping, or replacement are all your decisions to make. |

## Red Flags

- Recommending "debounce the handler" without measuring which phase (INPUTDELAY / PROCESSING / PRESENTATION) dominates.
- Assuming the handler author can't change third-party code.
- Naming a fix without confirming `inpel` (which element) and `inpls` (which load state).
- Stopping at "DOM is large" without measuring the size and the layout duration.
- Reporting INP in ms without reporting which phase owns the time.

## Summary format

After running all calls, print `✅ INP diagnosed: [BOTTLENECK PHASE] is [X]% of INP — [one-sentence cause]`

```
INP diagnosis for [PAGE]:
- p75: [VALUE]ms ([RATING]) | Mobile: [VALUE]ms | Desktop: [VALUE]ms
- Distribution: [X]% good / [Y]% needs improvement / [Z]% poor
- Slow interaction element: [CSS SELECTOR]
- Responsible script: [LOAF URL] ([first-party/third-party])
- Load state: [VALUE] — [EXPLANATION]
- Bottleneck phase: [PHASE] ([X]% of total INP)
  - INPUTDELAY: [VALUE]ms ([X]%)
  - PROCESSING: [VALUE]ms ([X]%)
  - PRESENTATION: [VALUE]ms ([X]%)
- Trend (7d): [improving/stable/regressing] ([X]% change)
- Distribution shape: [observation — what it means for this diagnosis] (only if histogram was used)
- Chrome should investigate: [SPECIFIC TRACE GOAL FROM BOTTLENECK SECTION]
```
