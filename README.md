# site-clone — Claude Code Website Cloning & Rebuilding Suite

> **Two skills, one workflow**: Clone any website → Rebuild as your own brand site.
>
> `/site-clone` — scrape, download, validate. `/site-rebuild` — analyze, extract, generate.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.ai/code)

---

## Quick Start

```bash
# Install both skills
git clone https://github.com/fanke009527-debug/site-clone.git ~/.claude/skills/site-clone
ln -s ~/.claude/skills/site-clone/site-rebuild ~/.claude/skills/site-rebuild

# Or on Windows (PowerShell):
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\site-rebuild" -Target "$env:USERPROFILE\.claude\skills\site-clone\site-rebuild"
```

Then in Claude Code:
```
/site-clone https://example.com/landing-page    # Step 1: clone
/site-rebuild                                    # Step 2: rebuild as brand site
```

---

## Skill 1: `/site-clone` — Website Cloning (v1.2)

| Step | Action |
|------|--------|
| 0. Pre-flight | Check accessibility, detect framework, auth/bot walls |
| 1. Setup | Create clone directory with domain-named folder |
| 2. Capture | Interaction sweep (scroll→hover→click) + Shadow DOM + Performance API |
| 3. Save | Write HTML with correct UTF-8 encoding (no BOM) |
| 4. Verify | Check encoding integrity, confirm non-ASCII characters preserved |
| 5. Download | Performance API + HTML attr regex + **CSS url() scan** + **external CSS file scan** |
| 6. Rewrite | Dynamic prefix discovery + `<base href="./">` safety net + Cloudflare cleanup |
| 7. Compare | Byte-level HTML comparison with diff location reporting |
| 8. Server | Node.js zero-dependency server with charset-correct MIME types |
| 9. Validate | Console error detection → download 404s → iterate to zero errors |
| 10. Manifest | Generate `site-manifest.json` with full inventory |
| 11. Screenshot | Full-page visual comparison |
| 12. Sitemap | Extract internal links + try sitemap.xml/robots.txt |

### v1.2 — CSS url() download gap fixed

The #1 silent asset loss was CSS `url()` references — background images in inline styles and fonts declared in external `.css` files (e.g. `@font-face { src: url(...) }`). Neither HTML attribute scanning nor Performance API reliably caught these.

**Step 5 now has two additional passes:**
- **5a**: Scans inline `<style>` blocks and `style=""` attributes for `url()` references
- **5d**: After downloading, scans every downloaded `.css` file for `url()` references and fetches any assets still missing

---

## Skill 2: `/site-rebuild` — Clone → Brand Site

Takes the output of `/site-clone` and rebuilds it as a config-driven Astro site.

| Phase | What it does |
|-------|-------------|
| 1. Inventory | Read manifest, screenshot the clone, report scope |
| 2. Topology | LLM visually maps page structure → `PAGE_TOPOLOGY.md` |
| 3. Design Tokens | `getComputedStyle()` on key elements → `design-tokens.json` |
| 4. Component Specs | Per-section HTML + CSS + assets + behavior → `specs/` directory |
| 5. Astro Generation | Config-driven `.astro` components with brand customization |
| 6. Validation | Build check, screenshot comparison, responsive testing |

### Design philosophy

**Clone is scriptable. Rebuild is not.** Rebuild requires visual judgment — which section is the Hero, where component boundaries fall, which color is the brand primary. The skill guides the LLM through this analysis and writes every decision to files before generating code, so you can review and correct.

### Output

```
brand-site/
├── src/
│   ├── config.ts              # Edit this to re-brand
│   ├── pages/index.astro      # Composes all sections
│   ├── layouts/BaseLayout.astro
│   ├── components/            # One .astro per section
│   └── styles/global.css
├── public/                    # Assets copied from clone
└── astro.config.ts
```

---

## How They Chain Together

```
User: "clone https://example.com and make it my brand site"

/site-clone:
  example.com/ → pre-flight → interaction sweep → download 77 files
  → rewrite paths → zero console errors → manifest + screenshot

/site-rebuild:
  reads clone output → maps 8 sections → extracts design tokens
  → writes 8 component specs → generates Astro project
  → `npx astro dev` running at localhost:4321

User edits src/config.ts → done.
```

---

## Compatibility

| AI Tool | site-clone | site-rebuild |
|---------|-----------|-------------|
| **Claude Code** | Full (Playwright + Performance API) | Full (needs browser for screenshot + computed styles) |
| **Codex CLI** | Static mode only | Partial (no browser for visual analysis) |
| **Cursor** | Static mode only | Partial |
| **aider / terminal AI** | Static mode only | Partial |

---

## File Structure

```
site-clone/
├── SKILL.md                  # /site-clone skill definition
├── README.md                 # This file
├── LICENSE                   # MIT
├── site-rebuild/
│   └── SKILL.md              # /site-rebuild skill definition
└── .gitignore
```

After running both skills:

```
site-clones/example.com/       # Clone output
├── index.html
├── server.js
├── site-manifest.json
├── PAGE_TOPOLOGY.md           # Added by site-rebuild
├── design-tokens.json         # Added by site-rebuild
├── specs/                     # Added by site-rebuild
├── images/ videos/ fonts/ ...

brand-site/                    # Generated Astro project
├── src/ ...
└── public/ ...
```

---

## Test Cases

**nudot.com.tw** — Chinese content, GSAP animations, Three.js 3D cube, 12 videos, Cloudflare protection.

| Phase | Tool | Result |
|-------|------|--------|
| Clone | /site-clone | 77 files, 19.2 MB, 0 console errors, byte-exact HTML |
| Rebuild | /site-rebuild | 5,535-line Astro site, 26/26 validation checks passed |

---

## License

MIT
