# site-clone — Claude Code Website Cloning Skill

> One-shot website cloning skill for Claude Code. Navigate → capture → download → rewrite → validate → done.
> Produces a fully offline copy of any web page with **zero console errors**.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)

## What it does (v1.1.0)

| Step | Action | New in v1.1.0 |
|------|--------|-----------|
| 0. Pre-flight | Check accessibility, detect framework, auth/bot walls | **NEW** |
| 1. Setup | Create clone directory with domain-named folder | |
| 2. Capture | Navigate + **interaction sweep** (scroll→hover→click) + Shadow DOM + Performance API | **Enhanced** |
| 3. Save | Write HTML with correct UTF-8 encoding (no BOM) | |
| 4. Verify | Check encoding integrity, confirm non-ASCII characters preserved | |
| 5. Download | Performance API resource list as ground truth + regex for remaining data-attributes | |
| 6. Rewrite | **Dynamic prefix discovery** (no hardcoded dirs) + `<base href="./">` safety net + Cloudflare cleanup | **Rewritten** |
| 7. Compare | Byte-level HTML comparison with diff location reporting | **Enhanced** |
| 8. Server | Start Node.js zero-dependency server with charset-correct MIME types | |
| 9. Validate | Console error detection → download 404s → iterate to zero errors | |
| 10. Manifest | Generate `site-manifest.json` with full inventory | |
| 11. Screenshot | Full-page visual comparison | |
| 12. Sitemap | Extract internal links + try sitemap.xml/robots.txt for multi-page discovery | **NEW** |

## Installation

```bash
git clone https://github.com/fanke009527-debug/site-clone.git ~/.claude/skills/site-clone
```

Requires:
- **Claude Code** CLI
- **Playwright MCP** (`npx @playwright/mcp`) — primary capture engine
- **bouncy MCP** — fast fallback capture (no browser needed)
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
扒站 https://example.com
```

## Compatibility

| AI Tool | Mode | Quality |
|---------|------|---------|
| **Claude Code** | Full (Playwright + Performance API) | Pixel-perfect match |
| **Codex CLI** | Static (curl + regex) | Functional, no JS rendering |
| **Cursor** | Static (curl + regex) | Functional, no JS rendering |
| **aider / terminal AI** | Static (curl + regex) | Functional, no JS rendering |
| **ChatGPT (Code Interpreter)** | Not supported | Not compatible |

For non-Claude-Code tools, assets loaded via JS (dynamic imports, Shadow DOM, CSS fonts, Web Workers) will be missing. See SKILL.md for all 15 documented limitations and workarounds.

## v1.1.0 — What's New

### Interaction Sweep (Step 2a)
Instead of a single scroll-to-bottom, the skill now performs a full interaction sweep: scroll in 10 increments to trigger all lazy-loaded content, hover navigation items to reveal dropdowns, click through carousels/tabs to capture all slides. This catches content that only appears on user interaction.

### Dynamic Path Rewriting (Step 6)
No more hardcoded directory patterns (`_nuxt/`, `images/`, `static/`, etc.). The skill auto-discovers all unique path prefixes from the downloaded asset list and generates rewrite rules dynamically. A `<base href="./">` tag is inserted as a safety net for any paths missed by rewriting.

### Framework Detection (Step 0)
Pre-flight check fingerprints the target site: Next.js, Nuxt, WordPress, Shopify, Astro. Each framework has specific asset path patterns and gotchas documented in SKILL.md. Bot detection and auth gates are flagged before any work begins.

### Cloudflare Artifact Cleanup (Step 6)
Automatically strips Cloudflare email obfuscation (`[email protected]`, `__cf_email__` spans), analytics beacons (`cf-beacon`), and Rocket Loader scripts. Emails are replaced with direct `mailto:` links.

### Sitemap Discovery (Step 12)
Extracts internal links from the cloned HTML and probes common sitemap locations (`sitemap.xml`, `robots.txt`). Pages are added to the manifest for follow-up cloning.

## How it was built

### The problem

Manually cloning a website with Claude Code required:
- Manually enumerating asset URLs from network logs
- Multiple rounds of downloading (static + dynamic/lazy-loaded assets always missed on first pass)
- Brittle, hardcoded regex path rewriting per site
- Manual console error inspection
- Silent UTF-8 encoding corruption for non-English content

A single-page clone took ~10 manual steps with 19 errors on first pass.

### The optimization

The skill replaces manual work with a 12-step automated pipeline:

```
Pass 0: Pre-flight → detect framework, check for auth/bot walls
Pass 1: Interaction sweep → capture all states → Shadow DOM → Performance API → save with verified UTF-8 → download ALL assets
Pass 2: Dynamic path rewriting → <base> safety net → Cloudflare cleanup → start server → console error inspection → download 404s
Pass 3: Re-validate → repeat until 0 errors → byte-level comparison with diff → manifest → sitemap
```

Key design decisions:

| Decision | Why |
|----------|-----|
| **Performance API as ground truth** | `performance.getEntriesByType('resource')` catches CSS @font-face, dynamic imports, workers — everything the browser loaded |
| **Interaction sweep before capture** | Scroll-then-hover-then-click triggers lazy content, dropdowns, carousel slides that a single snapshot misses |
| **Dynamic prefix discovery** | No hardcoded directory names — works on any site structure (Nuxt `_nuxt/`, Next `_next/`, Astro `_astro/`, WordPress `wp-content/`, custom) |
| Shadow DOM serialization | Web Components and shadow roots are recursively serialized into the HTML |
| Exhaustive attribute grep (16+ patterns) | Catches data-src, data-image, data-defer-src, srcset, poster, track, embed, object |
| UTF-8 encoding verification step | Double-encoding was the #1 silent bug; CJK/emoji would corrupt without warning |
| Byte-level HTML comparison with diff | One command catches encoding corruption, truncation, AND missing dynamic content; shows exact byte offset of first difference |
| Two capture modes | **Full** (Playwright, Claude Code) vs **Static** (curl, any AI tool) — documented tradeoffs |
| Framework-specific patterns | Next.js, Nuxt, WordPress, Shopify, Astro — each has known asset paths and gotchas |
| Known Limitations documented | 15 documented edge cases with workarounds so users know what to expect |

### Test cases

**Test 1:** `https://obsidianassembly.com/places` — Nuxt.js, custom fonts, WebP images, lazy-loaded route chunks.

| Metric | Pass 1 | Pass 2 | Final |
|--------|--------|--------|-------|
| Files downloaded | 28 | +19 | 49 |
| Console errors | 19 | 0 | 0 |

**Test 2:** `https://nudot.com.tw/` — Chinese content, GSAP animations, Three.js 3D cube, scrolling text marquee, 12 videos, Cloudflare protection.

| Metric | Pass 1 | Pass 2 | Final |
|--------|--------|--------|-------|
| Files downloaded | 37 | +40 | 77 |
| Total size | — | — | 19.2 MB |
| HTML match | Corrupted (UTF-16 LE) | Fixed (UTF-8) | Byte-exact |
| Console errors | — | — | 0 |
| Cloudflare artifacts | 3 removed | — | Clean |

Root causes discovered and fixed:
- **UTF-8 double-encoding** — PowerShell's default UTF-16 LE corrupted all Chinese characters. Fixed with `[System.IO.File]::WriteAllText` + `UTF8Encoding($false)`.
- **Cloudflare email obfuscation** — `email-decode.min.js` + `__cf_email__` spans hid real emails. Stripped both, replaced with direct `mailto:` links.
- **Cloudflare analytics** — `cf-beacon` script produced spurious 404s. Removed in cleanup step.
- **Hardcoded path patterns** — `_nuxt/` worked for Nuxt but not for Next.js or custom setups. Replaced with dynamic prefix discovery in v1.1.0.

## File structure

```
site-clone/
├── SKILL.md              # The skill definition (Claude Code loads this)
├── README.md             # This file
├── LICENSE               # MIT
└── .gitignore
```

After running, the clone output looks like:

```
site-clones/example.com/
├── index.html            # Rewritten HTML — byte-exact match after normalization
├── server.js             # Zero-dependency verification server
├── site-manifest.json    # Full inventory with file counts + discovered pages
├── sitemap.xml           # (if found) original sitemap
├── robots.txt            # (if found) original robots.txt
├── _nuxt/                # Framework build assets (Nuxt) — or _next/, _astro/, etc.
├── images/               # All images (preserved paths)
├── videos/               # Video assets
├── fonts/                # Font files
└── ...
```

## Prior art

- HTTrack — the classic website copier, but CLI-native and Claude Code-integrated
- gstack browse `archive` command — MHTML single-page snapshots
- Playwright's network interception API — the core capture mechanism
- ai-website-cloner-template — component-spec extraction and multi-platform skill syncing (complementary, for the rebuild phase)

## License

MIT
