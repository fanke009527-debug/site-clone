# site-clone — Website Cloning & Rebuilding Suite for Claude Code

> **Clone. Analyze. Rebuild.** A dual-skill pipeline that turns any website into a fully customizable, config-driven Astro site — in a single conversation.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)

```
  Target URL                    Clone Output                  Brand Site
  ─────────                    ────────────                  ──────────
  https://example.com  ──▶  site-clones/example.com/  ──▶  brand-site/
       │                        │                               │
       │  /site-clone           │  /site-rebuild                │
       │  • capture             │  • topology map               │
       │  • download            │  • design tokens              │
       │  • rewrite             │  • component specs            │
       │  • validate            │  • Astro generation           │
       │                        │                               │
       ▼                        ▼                               ▼
    77 files                zero console errors          npx astro dev @ :4321
```

---

## Overview

**site-clone** is a pair of Claude Code skills that together form a complete website cloning and rebranding pipeline. The first skill, `/site-clone`, performs a forensic-grade capture of a live website — HTML, CSS, JavaScript, fonts, images, and videos — and serves them locally with zero console errors. The second skill, `/site-rebuild`, reads that clone, analyzes its visual structure, extracts design tokens, and generates a brand-customizable Astro project.

**Why this exists.** Manually cloning a website for inspiration or rebranding is tedious and error-prone. Existing tools miss dynamically-loaded assets, break non-ASCII content, and leave you with a static snapshot you can't customize. This suite solves all three problems: exhaustive asset capture, byte-exact fidelity, and a generated codebase you actually own and can edit.

---

## Quick Start

### Installation

**macOS / Linux:**

```bash
git clone https://github.com/fanke009527-debug/site-clone.git ~/.claude/skills/site-clone
ln -s ~/.claude/skills/site-clone/site-rebuild ~/.claude/skills/site-rebuild
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/fanke009527-debug/site-clone.git $env:USERPROFILE\.claude\skills\site-clone
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\site-rebuild" -Target "$env:USERPROFILE\.claude\skills\site-clone\site-rebuild"
```

### Usage

Invoke directly in Claude Code:

```
/site-clone https://example.com/landing-page    # Step 1: clone the target site
/site-rebuild                                    # Step 2: rebuild as an Astro brand site
```

After `/site-rebuild` completes, customize your site by editing a single file:

```
brand-site/src/config.ts    ← brand name, colors, typography, content
```

---

## Skill 1: `/site-clone` — Website Cloning (v1.2)

A one-shot capture pipeline that produces a locally-served, pixel-accurate mirror of any website.

### Pipeline

| # | Stage | Description | Why It Matters |
|---|-------|-------------|----------------|
| 0 | **Pre-flight** | Accessibility check, framework fingerprinting, auth/bot-wall detection | Fails fast — avoids wasting time on unclonable targets |
| 1 | **Setup** | Create domain-named clone directory with mirrored path structure | Keeps assets organized exactly as the server expects |
| 2 | **Capture** | Interaction sweep (scroll → hover → click), Shadow DOM serialization, Performance API resource dump | Triggers lazy-loaded content and discovers assets that HTML scanners miss |
| 3 | **Save** | Write HTML with BOM-free UTF-8 encoding | Preserves Chinese, Japanese, emoji, and all non-ASCII content |
| 4 | **Verify** | Encoding integrity check | Confirms round-trip fidelity before proceeding |
| 5 | **Download** | Performance API entries + HTML attribute scanning + inline CSS `url()` extraction + external CSS file `url()` scanning | Four-pass asset discovery — catches background images, icon fonts, and `@font-face` sources |
| 6 | **Rewrite** | Dynamic prefix discovery, path rewriting, `<base href="./">` injection, Cloudflare artifact removal | Adapts to any framework's directory structure without hardcoded patterns |
| 7 | **Compare** | Byte-level HTML diff with location reporting | Detects corruption introduced during save/rewrite |
| 8 | **Server** | Zero-dependency Node.js static server with correct charset MIME types | Serves the clone exactly as a real web server would |
| 9 | **Validate** | Console error detection → download missing assets → iterate to zero errors | Converges to a clean, warning-free local clone |
| 10 | **Manifest** | Generate `site-manifest.json` with full file inventory and sizing | Feeds structured data into `/site-rebuild` |
| 11 | **Screenshot** | Full-page visual comparison against original | Catches layout differences that byte-level checks miss |
| 12 | **Sitemap** | Internal link extraction, `sitemap.xml` and `robots.txt` probing | Discovers multi-page sites beyond the entry URL |

### Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| **Performance API as primary asset source** | `performance.getEntriesByType('resource')` captures everything the browser actually loaded — including dynamically-injected `<script>`, lazy images, and XHR-fetched assets that regex scanning misses |
| **Four-pass CSS asset discovery** | Inline `<style>` blocks, `style=""` attributes, and external `.css` files each hide `url()` references. Only scanning all three catches everything |
| **Dynamic path prefix discovery** | Instead of hardcoding `/_nuxt/`, `/static/`, etc., the pipeline auto-discovers prefixes from the asset list — works with any framework |
| **BOM-free UTF-8** | PowerShell defaults to UTF-16 LE. Using `[System.IO.File]::WriteAllText` with `UTF8Encoding($false)` prevents BOM corruption that breaks CSS and JS |
| **Zero-dependency server** | A 30-line Node.js `http` server with correct MIME types and charset headers — no `npm install` needed |
| **Convergent validation loop** | Rather than a single pass, the validator re-checks console errors after each fix until clean — handles cascading 404s |

---

## Skill 2: `/site-rebuild` — Clone → Brand Site

Transforms a cloned website into a config-driven Astro project with full brand customization.

### Philosophy

> **Clone is scriptable. Rebuild is not.**

Downloading files can be automated. But identifying which section is the Hero, where component boundaries fall, and which color is the brand primary — these require visual judgment. `/site-rebuild` guides the LLM through this analysis, writing every decision to reviewable files before generating any code.

### Pipeline

| # | Phase | Output | Description |
|---|-------|--------|-------------|
| 1 | **Inventory** | — | Reads manifest, serves the clone, takes a full-page screenshot, reports scope to the user |
| 2 | **Topology** | `PAGE_TOPOLOGY.md` | LLM analyzes the screenshot and HTML to identify semantic sections (Hero, Features, CTA, Footer...) with DOM selectors, heights, and interaction notes |
| 3 | **Design Tokens** | `design-tokens.json` | `getComputedStyle()` extraction on each section's root element — colors, typography, spacing, radii, shadows |
| 4 | **Component Specs** | `specs/{section}.md` | Per-section HTML structure, CSS rules, asset mappings, and behavior notes |
| 5 | **Astro Generation** | `brand-site/` | Generates a complete Astro project: `config.ts` for branding, one `.astro` component per section, `BaseLayout.astro` shell, `global.css` with CSS custom properties |
| 6 | **Validation** | — | Build check, screenshot comparison, responsive testing at 375px / 768px / 1200px, config audit |

### Generated Project Structure

```
brand-site/
├── src/
│   ├── config.ts              # ← Edit this to re-brand (colors, fonts, content)
│   ├── pages/
│   │   └── index.astro        # Composes all sections in order
│   ├── layouts/
│   │   └── BaseLayout.astro   # HTML shell, meta tags, font loading, analytics
│   ├── components/
│   │   ├── Nav.astro
│   │   ├── Hero.astro
│   │   ├── Features.astro     # One component per topology section
│   │   ├── ...
│   │   └── Footer.astro
│   └── styles/
│       └── global.css         # CSS custom properties + Tailwind imports
├── public/
│   ├── images/                # Copied from clone
│   ├── videos/
│   └── fonts/
└── astro.config.ts
```

### Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Config-driven** | All brand values in `config.ts` — zero hardcoded strings in components |
| **Component-per-section** | Each semantic section is its own `.astro` file — add, remove, or reorder at will |
| **Scoped styles** | Each component carries its CSS in a `<style>` block — no global namespace pollution |
| **Reviewable decisions** | Topology, tokens, and specs are written to files before code generation — user approves before anything is built |
| **Validation gate** | Build must pass, screenshots must match, responsive breakpoints must work |

---

## How They Chain

```
User types: "clone https://example.com and make it my brand site"

┌─────────────────────────────────────────────────────────┐
│ /site-clone                                              │
│                                                          │
│  example.com/  →  pre-flight check (framework: Nuxt)     │
│               →  interaction sweep (scroll, hover, click) │
│               →  capture 84 network requests              │
│               →  download 77 assets (19.2 MB)             │
│               →  rewrite paths (dynamic prefix: /_nuxt/)  │
│               →  start server @ localhost:8765            │
│               →  validate: 0 console errors               │
│               →  manifest + screenshot                    │
│                                                          │
│  Output: site-clones/example.com/                        │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│ /site-rebuild                                            │
│                                                          │
│  reads clone  →  maps 8 semantic sections                │
│              →  extracts design tokens (colors, fonts)    │
│              →  writes 8 component specs                  │
│              →  generates Astro project                   │
│              →  npx astro build: ✅ pass                  │
│              →  screenshot comparison: ✅ match            │
│                                                          │
│  Output: brand-site/ (npx astro dev @ localhost:4321)    │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
          User edits src/config.ts → done.
```

---

## Compatibility

| Environment | `/site-clone` | `/site-rebuild` |
|-------------|:---:|:---:|
| **Claude Code** (desktop / CLI / IDE) | Full | Full |
| **Codex CLI** | Static mode only | Partial |
| **Cursor** | Static mode only | Partial |
| **aider / terminal agents** | Static mode only | Partial |

**Static mode** falls back to `fetch` + `scrape` (via bouncy MCP) for HTML and visible assets — no JavaScript execution, no lazy-loaded content, no Performance API. Suitable for simple static sites.

---

## Verified Test Cases

### nudot.com.tw

A complex commercial site with Chinese content, GSAP scroll animations, Three.js 3D rendering, 12 background videos, and Cloudflare protection.

| Metric | Result |
|--------|--------|
| Assets captured | 77 files, 19.2 MB |
| Console errors | 0 |
| HTML fidelity | Byte-exact after rewrite |
| Framework detected | Nuxt 3 |
| Clone time | ~2 minutes |
| Rebuild output | 5,535 lines of Astro, 26/26 validation checks passed |

### obsidianassembly.com

A modern documentation site built with Next.js, testing static generation patterns and MDX routing.

| Metric | Result |
|--------|--------|
| Assets captured | 100% (no missing resources) |
| Console errors | 0 |
| Framework detected | Next.js (ISR) |

---

## Repository Structure

```
site-clone/
├── SKILL.md                   # /site-clone skill definition (831 lines, v1.2)
├── README.md                  # This file
├── LICENSE                    # MIT
├── site-rebuild/
│   └── SKILL.md               # /site-rebuild skill definition (306 lines, v1.0)
└── .gitignore
```

After running both skills, your working directory will contain:

```
site-clones/{domain}/           # Clone output (auto-created)
├── index.html                  # Rewritten HTML with local paths
├── server.js                   # Zero-dependency static server
├── site-manifest.json          # Full asset inventory
├── PAGE_TOPOLOGY.md            # Semantic section map (from /site-rebuild)
├── design-tokens.json          # Extracted visual design tokens (from /site-rebuild)
├── specs/                      # Per-section component specs (from /site-rebuild)
│   ├── Nav.md
│   ├── Hero.md
│   └── ...
├── images/ videos/ fonts/ ...  # Downloaded assets

brand-site/                     # Generated Astro project (from /site-rebuild)
├── src/
│   ├── config.ts
│   ├── pages/index.astro
│   ├── layouts/BaseLayout.astro
│   ├── components/
│   └── styles/global.css
├── public/                     # Copied assets
└── astro.config.ts
```

---

## Limitations

- **Auth-gated sites** cannot be cloned (requires valid session credentials)
- **CAPTCHA-protected pages** will block automated access
- **Complex WebGL / Three.js scenes** are captured as static snapshots — interactive 3D state is not preserved
- **Streaming / WebSocket content** is not captured (only the initial render)
- **E-commerce logic** (carts, checkout, user accounts) is not recreated — the generated site is static
- `/site-rebuild` requires Claude Code (needs a browser for visual analysis)

---

## License

MIT — see [LICENSE](LICENSE) for details.
