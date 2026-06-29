# site-clone — Claude Code Website Cloning Skill

> One-shot website cloning skill for Claude Code. Navigate → capture → download → rewrite → validate → done.
> Produces a fully offline, byte-exact copy of any web page with **zero console errors**.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)

## What it does

| Step | Action | New in v2 |
|------|--------|-----------|
| 1. Setup | Create clone directory with domain-named folder | |
| 2. Capture | Navigate + scroll + capture DOM with Shadow DOM serialization + Performance API resource list | **Enhanced** |
| 3. Save | Write HTML with correct UTF-8 encoding (no BOM) — **#1 failure point fixed** | |
| 4. Verify | Check encoding integrity, confirm non-ASCII characters preserved | **NEW** |
| 5. Download | Performance API resource list as ground truth + regex for remaining data-attributes | **Enhanced** |
| 6. Rewrite | Systematic regex path replacement (not hardcoded per-directory) | **Enhanced** |
| 7. Compare | Byte-level HTML comparison: live vs local after normalization | **NEW** |
| 8. Server | Start Node.js zero-dependency server with charset-correct MIME types | |
| 9. Validate | Console error detection → download 404s → iterate to zero errors | |
| 10. Manifest | Generate `site-manifest.json` with full inventory | |
| 11. Screenshot | Full-page visual comparison | |

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
| **Claude Code** | Full (Playwright + Performance API) | Byte-exact match |
| **Codex CLI** | Static (curl + regex) | Functional, no JS rendering |
| **Cursor** | Static (curl + regex) | Functional, no JS rendering |
| **aider / terminal AI** | Static (curl + regex) | Functional, no JS rendering |
| **ChatGPT (Code Interpreter)** | Not supported | Not compatible |

For non-Claude-Code tools, assets loaded via JS (dynamic imports, Shadow DOM, CSS fonts, Web Workers) will be missing. See SKILL.md for all 12 documented limitations and workarounds.

## How it was built

### The problem

Manually cloning a website with Claude Code required:
- Manually enumerating asset URLs from network logs
- Multiple rounds of downloading (static + dynamic/lazy-loaded assets always missed on first pass)
- Brittle regex path rewriting
- Manual console error inspection

A single-page clone took ~10 manual steps with 19 errors on first pass.

### The optimization

The skill replaces manual work with an 11-step automated pipeline:

```
Pass 1: Setup → Capture (with Shadow DOM + Performance API) → Save (verified UTF-8) → Download ALL assets
Pass 2: Start server → Console error inspection → Extract 404 URLs → Download missing assets
Pass 3: Re-validate → Repeat until 0 errors → Byte-level HTML comparison → Manifest
```

Key design decisions:

| Decision | Why |
|----------|-----|
| **Performance API as ground truth** | `performance.getEntriesByType('resource')` catches CSS @font-face, dynamic imports, workers — everything the browser loaded |
| Shadow DOM serialization | Web Components and shadow roots are recursively serialized into the HTML |
| Exhaustive attribute grep (12+ patterns) | Catches data-src, data-image, data-defer-src, srcset, poster, track, embed, object |
| UTF-8 encoding verification step | Double-encoding was the #1 silent bug; CJK/emoji would corrupt without warning |
| Byte-level HTML comparison | One command catches encoding corruption, truncation, AND missing dynamic content |
| Two capture modes | **Full** (Playwright, Claude Code) vs **Static** (curl, any AI tool) — documented tradeoffs |
| Known Limitations documented | 12 documented edge cases with workarounds so users know what to expect |

### Test cases

**Test 1:** `https://obsidianassembly.com/places` — Nuxt.js, custom fonts, WebP images, lazy-loaded route chunks.

| Metric | Pass 1 | Pass 2 | Final |
|--------|--------|--------|-------|
| Files downloaded | 28 | +19 | 49 |
| Console errors | 19 | 0 | 0 |

**Test 2:** `https://nudot.com.tw/` — Chinese content, GSAP animations, Three.js 3D cube, scrolling text marquee, 12 videos.

| Metric | Pass 1 | Pass 2 | Final |
|--------|--------|--------|-------|
| Files downloaded | 37 | +40 | 77 |
| Total size | — | — | 19.2 MB |
| HTML match | Corrupted | Fixed | Byte-exact |
| Console errors | — | — | 0 |

Root cause discovered in Test 2: **UTF-8 double-encoding** — the HTML was saved with PowerShell's default encoding (UTF-16 LE), corrupting all Chinese characters. Fixed by using `[System.IO.File]::WriteAllText` with explicit UTF-8 encoding in Step 3, and adding encoding verification in Step 4.

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
├── site-manifest.json    # Full inventory with file counts by type
├── images/               # All images (preserved paths)
│   ├── home/
│   ├── nav/
│   └── ...
├── videos/               # Video assets
├── fonts/                # Font files
└── ...
```

## Prior art

- HTTrack — the classic website copier, but CLI-native and Claude Code-integrated
- gstack browse `archive` command — MHTML single-page snapshots
- Playwright's network interception API — the core capture mechanism

## License

MIT
