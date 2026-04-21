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
/plugin install cwv-superpowers@cwv-superpowers
```

Then set up CoreDash MCP if you haven't already:

```bash
claude mcp add --transport http coredash https://app.coredash.app/api/mcp \
  --header "Authorization: Bearer cdk_YOUR_API_KEY"
```

Replace `cdk_YOUR_API_KEY` with your key from CoreDash → Project Settings → API Keys (MCP).

Restart Claude Code after installing so the SessionStart hook fires on a fresh conversation.

### Cursor

The repo ships a Cursor plugin manifest at `.cursor-plugin/plugin.json` pointing at the same skill set. Install via Cursor's plugin UI from the GitHub repo URL.

### Gemini CLI

The repo ships a Gemini extension at `gemini-extension.json` with `GEMINI.md` auto-loading the orchestrator skill. Install via:

```bash
gemini extensions install https://github.com/corewebvitals/cwv-superpowers
```

### Other MCP-compatible clients

The CoreDash MCP server works with any client that supports HTTP MCP servers:

| Setting | Value |
|---|---|
| Endpoint | `https://app.coredash.app/api/mcp` |
| Header | `Authorization: Bearer cdk_YOUR_API_KEY` |

Ask the agent to "set up CoreDash" — the `setting-up-coredash` skill walks through detailed per-client setup (Claude Desktop, Windsurf, Gemini CLI, etc.).

### Verify installation

Ask your agent: **"What are my Core Web Vitals?"**

If CoreDash is connected, it will return your real LCP, INP, CLS, FCP, and TTFB data.

### Permission prompts

From v2.0.2 the plugin auto-approves the read-only tools it needs (`Read`, `Grep`, `Glob`, and the 3 CoreDash MCP queries) via a `PreToolUse` hook, so diagnostic runs don't flood you with prompts. Write/Edit/Bash (code fixes) and all Chrome DevTools tools (navigation, script eval) stay behind permission prompts by design.

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

Six peer skills: one orchestrator + five specialist skills. The orchestrator runs the full multi-step flow; the specialists can also be invoked directly.

```
cwv-superpowers/                        ← Repo root (git repo)
├── .claude-plugin/marketplace.json     ← Claude Code marketplace listing
├── .cursor-plugin/plugin.json          ← Cursor manifest
├── gemini-extension.json               ← Gemini CLI extension manifest
├── GEMINI.md                           ← Gemini auto-loaded context
├── .version-bump.json                  ← Single source of truth for version
├── scripts/bump-version.sh             ← Keeps all 10 version fields in sync
└── plugins/cwv-superpowers/
    ├── .claude-plugin/plugin.json      ← Claude Code plugin metadata
    ├── hooks/
    │   ├── hooks.json                  ← SessionStart + PreToolUse hook config
    │   ├── hooks-cursor.json           ← Cursor hook config
    │   ├── session-start               ← Injects capability-tier preamble
    │   └── pre-tool-use-allow          ← Auto-approves read-only tools
    └── skills/
        ├── cwv-superpower/             ← Orchestrator (Step 0–5 flow)
        │   ├── SKILL.md
        │   └── templates/
        │       ├── report-rum.html     ← RUM-only report template
        │       └── report-full.html    ← Full report (filmstrip, waterfall)
        ├── diagnosing-lcp/SKILL.md     ← LCP diagnosis (phases + attribution)
        ├── diagnosing-inp/SKILL.md     ← INP diagnosis (phases + LOAF scripts)
        ├── diagnosing-cls/SKILL.md     ← CLS diagnosis (cause pattern matching)
        ├── tracing-with-chrome/SKILL.md ← Chrome tracing (per-phase investigation)
        └── setting-up-coredash/SKILL.md ← CoreDash MCP installation guide
```

**When each skill activates:**

- `cwv-superpower` — "find my biggest CWV issue", "audit this site", multi-metric analysis.
- `diagnosing-lcp` / `diagnosing-inp` / `diagnosing-cls` — the user names one specific metric.
- `tracing-with-chrome` — the user has a URL + metric + suspected cause and wants a lab trace.
- `setting-up-coredash` — connecting CoreDash MCP from scratch.

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
/plugin marketplace update cwv-superpowers
/plugin install cwv-superpowers@cwv-superpowers
```

Restart Claude Code after updating.

## Contributing

Issues and PRs welcome at [github.com/corewebvitals/cwv-superpowers](https://github.com/corewebvitals/cwv-superpowers).

### For maintainers

Versions across all 10 manifests (Claude Code, Cursor, Gemini CLI, plugin.json, and all 6 SKILL.md files) are kept in sync via one script:

```bash
./scripts/bump-version.sh --check      # show current versions + detect drift
./scripts/bump-version.sh --audit      # check + scan repo for stray version strings
./scripts/bump-version.sh 2.1.0        # bump every declared file to 2.1.0
```

The declared file list lives in `.version-bump.json`. To add a new version-bearing file, add its path + JSON field (or `"format": "yaml-frontmatter"` for SKILL.md frontmatter).

## License

MIT
