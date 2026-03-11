# CWV Superpowers

Core Web Vitals diagnosis and fixing skills for AI coding agents — powered by [CoreDash](https://coredash.app) real user monitoring.

**What it does:** Connects your real user data (LCP, INP, CLS) to your AI coding agent. The agent finds the worst-performing pages, identifies the exact element and bottleneck phase causing the issue, traces the root cause in Chrome, and either fixes the code or generates a detailed HTML report — all from a single conversation.

## What You Get

| Capability | What it does |
|---|---|
| **Automated discovery** | Finds your worst pages and metrics across millions of real page loads |
| **Phase breakdown** | Splits LCP into TTFB / Load Delay / Load Time / Render Delay — names the bottleneck |
| **INP attribution** | Identifies the slow interaction element, the responsible script (LOAF), and the load state |
| **CLS cause matching** | Detects images without dimensions, font swaps, injected content, late-loading resources |
| **Chrome tracing** | Visits the page with mobile emulation and traces the exact bottleneck identified by RUM |
| **Code fixes** | Makes the minimal code change — names the file, the line, the element |
| **HTML reports** | Generates interactive reports with filmstrip, waterfall, breakdown charts, root cause analysis |

## Requirements

- **[CoreDash](https://coredash.app) account** with real user data flowing (free tier works)
- **CoreDash API key** (generated in Project Settings → API Keys)
- **Claude Code** — `npm install -g @anthropic-ai/claude-code`
- **Chrome** (optional but recommended) — run Claude Code with `claude --chrome` for full tracing

## Installation

### Claude Code

```bash
# Add the marketplace
/plugin marketplace add corewebvitals/cwv-superpowers

# Install the plugin
/plugin install cwv-superpowers@cwv-superpower
```

Then set up CoreDash MCP if you haven't already:

```bash
claude mcp add --transport http coredash https://app.coredash.app/api/mcp \
  --header "Authorization: Bearer cdk_YOUR_API_KEY"
```

Replace `cdk_YOUR_API_KEY` with your key from CoreDash → Project Settings → API Keys (MCP).

### Cursor

```
/plugin-add cwv-superpowers
```

### Other MCP-compatible clients

The CoreDash MCP server works with any client that supports HTTP MCP servers:

| Setting | Value |
|---|---|
| Endpoint | `https://app.coredash.app/api/mcp` |
| Header | `Authorization: Bearer cdk_YOUR_API_KEY` |

See `plugins/cwv-superpowers/skills/cwv-superpower/modules/setup.md` for detailed setup instructions per client (Claude Desktop, Windsurf, Gemini CLI).

### Verify installation

Ask your agent: **"What are my Core Web Vitals?"**

If CoreDash is connected, it will return your real LCP, INP, CLS, FCP, and TTFB data.

## Usage

Start Claude Code (with Chrome for full analysis):

```bash
claude --chrome
```

Then just ask:

- **"Find my biggest CWV issue and fix it"** — automated discovery + diagnosis + fix
- **"My product pages are slow"** — targeted diagnosis on a page section
- **"LCP on /product/shoes-42 is bad"** — drill into a specific URL
- **"Generate a report"** — get an interactive HTML report with all findings

The skill handles capability detection automatically. It works with:

| Chrome | CoreDash | What you get |
|---|---|---|
| Yes | Yes | Full analysis — RUM finds the issue, Chrome explains why, code fix + visual report |
| No | Yes | RUM diagnosis — element, bottleneck phase, trend, code fix (no Chrome visuals) |
| Yes | No | Lab-only audit — Chrome trace without real user context (setup guide offered) |
| No | No | Setup wizard — guides you through installing both |

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  1. Discovery                                               │
│     CoreDash → worst pages, worst metrics, distributions    │
├─────────────────────────────────────────────────────────────┤
│  2. Diagnosis                                               │
│     LCP: TTFB / LOADDELAY / LOADTIME / RENDERDELAY         │
│     INP: INPUTDELAY / PROCESSING / PRESENTATION             │
│     CLS: 5 cause patterns (images, fonts, injected, ...)   │
├─────────────────────────────────────────────────────────────┤
│  3. Chrome Trace                                            │
│     Mobile emulation (Fast 3G, 4x CPU slowdown)            │
│     Investigates ONLY the bottleneck phase from step 2     │
├─────────────────────────────────────────────────────────────┤
│  4. Root Cause                                              │
│     Names the element, the cause, both evidence sources     │
├─────────────────────────────────────────────────────────────┤
│  5. Output                                                  │
│     Code fix  /  HTML report  /  Both                       │
└─────────────────────────────────────────────────────────────┘
```

## Skill Structure

```
plugins/cwv-superpowers/
├── .claude-plugin/
│   └── plugin.json       ← Plugin metadata
└── skills/cwv-superpower/
    ├── SKILL.md           ← Main orchestrator (start here)
    ├── modules/
    │   ├── setup.md       ← CoreDash MCP installation guide
    │   ├── lcp.md         ← LCP diagnosis (phases + attribution)
    │   ├── inp.md         ← INP diagnosis (phases + LOAF scripts)
    │   ├── cls.md         ← CLS diagnosis (cause pattern matching)
    │   └── chrome.md      ← Chrome tracing (per-phase investigation)
    └── templates/
        ├── report-rum.html  ← RUM-only report template
        └── report-full.html ← Full report (filmstrip, waterfall, tabs)
```

## Example Output

**Root cause statement:**

> **Root cause:** The LCP image `div.hero-banner > img.product-main` on `/product/running-shoes-42` is discovered 1,980ms late because it lacks a preload hint and has no `fetchpriority="high"`.
>
> **Evidence from real users (CoreDash):** LCP is 3,820ms (poor) on mobile, p75. LOADDELAY is the bottleneck at 1,980ms (52% of total LCP). The image priority state is 3 (not preloaded). LCP has worsened by 340ms over 7 days.
>
> **Evidence from lab trace (Chrome):** The network waterfall shows a 1,940ms gap between HTML first byte and the hero image request. The image is referenced only in CSS `background-image`, invisible to the preload scanner. No `<link rel="preload">` exists in `<head>`.

**Code fix:**

```html
<!-- Add to <head> -->
<link rel="preload" href="/images/hero.jpg" as="image" fetchpriority="high">

<!-- Change the img element -->
<img src="/images/hero.jpg" alt="Hero" fetchpriority="high">
```

## Updating

```bash
/plugin update cwv-superpowers
```

## Contributing

Issues and PRs welcome at [github.com/corewebvitals/cwv-superpowers](https://github.com/corewebvitals/cwv-superpowers).

## License

MIT
