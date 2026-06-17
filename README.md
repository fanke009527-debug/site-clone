# site-clone — Claude Code Website Cloning Skill

> One-shot website cloning skill for Claude Code. Navigate → capture → download → rewrite → validate → done. Produces a fully offline copy of any web page with **zero console errors**.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)

## What it does

| Step | Action |
|------|--------|
| 1. Capture | Navigate to URL with Playwright, record ALL network requests in one pass |
| 2. Download | Auto-download every asset (CSS, JS, images, fonts) preserving directory structure |
| 3. Rewrite | Replace absolute paths with relative paths — surgical, domain-scoped only |
| 4. Validate | Start local server → detect 404s → download missing → iterate to zero errors |
| 5. Manifest | Generate `site-manifest.json` with full inventory |

## Installation

```bash
# Clone into Claude Code skills directory
git clone https://github.com/YOUR_USERNAME/site-clone.git ~/.claude/skills/site-clone
```

Requires:
- **Claude Code** CLI
- **Playwright MCP** (`npx @playwright/mcp`) — installed and connected
- **Node.js** — for the local verification server

## Usage

In Claude Code, just say:

```
/site-clone https://example.com/landing-page
```

Or use natural language:

```
clone this site: https://example.com
复刻这个网站
Save this page offline
```

The skill will:

1. Open the page and capture all resources
2. Download everything to `E:\Projects\claude-code\site-clones\{domain}\`
3. Start a local server at `http://localhost:8765/`
4. Iterate until console shows 0 errors
5. Open your browser to the local copy

## How it was built

### The problem

Manually cloning a website with Claude Code required:

- Manually enumerating asset URLs from network logs
- Multiple rounds of downloading (static + dynamic/lazy-loaded assets always missed on first pass)
- Brittle regex path rewriting
- Manual console error inspection

A single-page clone took ~10 manual steps with 19 errors on first pass.

### The optimization

The skill replaces that with a 3-pass automated loop:

```
Pass 1: Playwright navigation → capture ALL network requests → download everything
Pass 2: Console error inspection → extract 404 URLs → download missing assets
Pass 3: Re-validate → repeat until 0 errors
```

Key design decisions:

| Decision | Why |
|----------|-----|
| Network log as source of truth | Eliminates manual URL enumeration; captures dynamic JS-loaded assets |
| Iterative 404 repair | Lazy-loaded route chunks and Vue/React component images only fire after hydration — can't get them on pass 1 |
| Domain-scoped path rewriting | `src="/images/x.png"` → `src="./images/x.png"` only for target domain; external CDN URLs untouched |
| `<base href="./">` safety net | Catches edge cases the regex misses |
| Node.js zero-dependency server | No `npm install` needed — just `node server.js` |

### First test case

**Target:** `https://obsidianassembly.com/places` — a Nuxt.js site with custom fonts, WebP images, lazy-loaded route chunks.

| Metric | Pass 1 | Pass 2 | Final |
|--------|--------|--------|-------|
| Files downloaded | 28 | +19 | **49** |
| Console errors | 19 | 0 | **0** |
| Missing assets | 19 | 0 | **0** |

Only permanent failure: `/videos/walk-through.mp4` — 404 on the original server too.

## File structure

```
site-clone/
├── SKILL.md              # The skill definition (Claude Code loads this)
├── README.md             # This file
└── LICENSE               # MIT
```

After running, the clone output looks like:

```
site-clones/example.com/
├── index.html            # Rewritten HTML with relative paths
├── server.js             # Zero-dependency verification server
├── site-manifest.json    # Full inventory
├── _nuxt/                # JS/CSS bundles (preserved paths)
├── images/               # All images (preserved paths)
├── fonts/                # Font files
└── ...
```

## Skills that pair well with this

- **[gstack browse](https://github.com/garrytan/gstack)** — Archive single pages as MHTML. site-clone is more thorough (multi-pass, zero-error guarantee) but gstack is faster for quick snapshots.
- **[web-scraper](https://github.com/yfe404/web-scraper)** — Adaptive 6-phase scraping with anti-bot detection. Use web-scraper for protected sites; use site-clone for clean public pages.
- **[browserbase/skills](https://github.com/browserbase/skills)** — 11 browser automation skills. site-clone is purpose-built for cloning; browserbase is a broader toolkit.

## Prior art

This skill was inspired by:
- HTTrack — the classic website copier, but CLI-native and Claude Code-integrated
- gstack browse `archive` command — MHTML single-page snapshots
- Playwright's network interception API — the core capture mechanism

## License

MIT
