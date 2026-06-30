---
name: site-rebuild
description: |
  Rebuild a cloned website as a brand-customizable Astro site.
  Reads the output of /site-clone (HTML + assets + manifest) and
  generates config-driven components with design tokens.
  Use after /site-clone, or when asked to "turn this clone into a brand site",
  "rebuild this as my own site", "生成品牌站".
triggers:
  - rebuild this site
  - turn this clone into a brand site
  - 生成品牌站
  - 重建为品牌站
  - make this my own site
allowed-tools:
  - Bash
  - PowerShell
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_take_screenshot
  - mcp__playwright__browser_console_messages
  - mcp__bouncy__fetch
  - mcp__bouncy__scrape
---

# Site Rebuild — Clone → Brand Site Pipeline

Takes a `/site-clone` output directory and rebuilds it as a config-driven, brand-customizable Astro site.

**Prerequisite:** Run `/site-clone <url>` first. This skill reads its output.

---

## Philosophy

Clone is scriptable. Rebuild is not. Rebuild requires visual judgment — which section is the Hero, which color is the brand primary, where the component boundaries fall. This skill guides the LLM through that judgment process, step by step.

**Every decision the LLM makes should be written to a file** (topology, tokens, specs) so the user can review and correct before any code is generated.

---

## Workflow

### Phase 1: Inventory — Know What We're Working With

**First, locate the clone directory.** The site-clone output lives at `E:\Projects\claude-code\site-clones\{domain}/`. If you ran /site-clone in the same session, the `$cloneDir` variable should still be in context. Otherwise:

```powershell
# Find the most recent clone
$cloneDir = Get-ChildItem -Path "E:\Projects\claude-code\site-clones" -Directory |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1 |
    ForEach-Object { $_.FullName }
Write-Host "Clone directory: $cloneDir"
```

Read the clone output to understand scope:

1. Read `$cloneDir/site-manifest.json` — file count, asset breakdown, discovered pages
2. Read `$cloneDir/index.html` — get a feel for size and structure (line count, major `<section>` tags)
3. Start the clone's server if not running: `node $cloneDir/server.js &`
4. Open `http://localhost:8765/` in Playwright and take a full-page screenshot
5. Report to user: "I have a [X]KB page with [Y] assets from {domain}. Ready to analyze structure."

```
mcp__playwright__browser_navigate → http://localhost:8765/
mcp__playwright__browser_take_screenshot → full page
```

### Phase 2: Page Topology — Map the Structure

LLM visually analyzes the screenshot and HTML to identify semantic sections. This is the most important step — everything downstream depends on getting the boundaries right.

**How to identify section boundaries:**
1. Look for `<section>`, `<header>`, `<footer>`, `<main>`, `<nav>` tags in HTML
2. Look for visual dividers in the screenshot: full-width color bands, spacing gaps, content shifts
3. Look for heading hierarchy (`<h1>` through `<h6>`) as section markers
4. Look for class names suggesting sections: `hero`, `features`, `cta`, `footer`, `about`, `testimonials`, `pricing`, `contact`

**Output: Write `$cloneDir/PAGE_TOPOLOGY.md`** with this structure:

```markdown
# Page Topology — {domain}

## Section Map

| # | Section Name | DOM Selector | Type | Est. Height | Interactive |
|---|-------------|-------------|------|-------------|-------------|
| 1 | Nav | `nav` or `header` | Navigation | ~80px | Hover dropdowns |
| 2 | Hero | `.hero` or first `<section>` | Hero | ~100vh | Scroll-triggered animation |
| 3 | ... | ... | ... | ... | ... |

## DOM Hierarchy
(ASCII tree showing parent-child section relationships)

## Key Observations
- Framework detected: (Nuxt / Next.js / WordPress / vanilla)
- Animation library: (GSAP / Framer Motion / none)
- Responsive breakpoints observed: (list)
- Dynamic content: (items that change on user interaction)
```

**Before continuing, show PAGE_TOPOLOGY.md to the user for approval.** They may want to rename, merge, or split sections.

### Phase 3: Design Token Extraction

Extract the visual design language into machine-readable tokens. This is done via Playwright — the LLM uses `browser_evaluate` to run `getComputedStyle()` on key elements identified in the topology.

**For each section identified in Phase 2, extract:**

```javascript
// Run in Playwright for each section's root element
() => {
  const el = document.querySelector('{section_selector}');
  if (!el) return null;
  const s = getComputedStyle(el);
  return JSON.stringify({
    // Typography
    fontFamily: s.fontFamily,
    fontSize: s.fontSize,
    fontWeight: s.fontWeight,
    lineHeight: s.lineHeight,
    letterSpacing: s.letterSpacing,
    // Colors
    color: s.color,
    backgroundColor: s.backgroundColor,
    // Spacing
    padding: s.padding,
    margin: s.margin,
    // Decoration
    borderRadius: s.borderRadius,
    borderWidth: s.borderWidth,
    boxShadow: s.boxShadow,
    // Layout
    display: s.display,
    flexDirection: s.flexDirection,
    maxWidth: s.maxWidth,
    gap: s.gap
  });
}
```

**Also capture global tokens from `:root` / `body`:**
- CSS custom properties (`--primary`, `--background`, etc.) — read from `getComputedStyle(document.documentElement)`
- Brand colors: sample the dominant colors from the logo area and hero section
- Font stack: extract from `body`, `h1`-`h6` computed styles

**Output: Write `$cloneDir/design-tokens.json`:**

```json
{
  "colors": {
    "background": "oklch(1 0 0)",
    "foreground": "oklch(0.145 0 0)",
    "primary": "oklch(0.205 0 0)",
    "accent": "oklch(0.488 0.243 264.376)",
    "muted": "oklch(0.97 0 0)"
  },
  "typography": {
    "heading": { "fontFamily": "Geist, sans-serif", "fontWeight": "700" },
    "body": { "fontFamily": "Geist, sans-serif", "fontWeight": "400", "fontSize": "16px" },
    "mono": { "fontFamily": "Geist Mono, monospace" }
  },
  "spacing": {
    "section": "80px",
    "content": "1200px",
    "gap": "24px"
  },
  "radius": {
    "sm": "4px",
    "md": "10px",
    "lg": "16px"
  }
}
```

### Phase 4: Component Spec Extraction

For each section in PAGE_TOPOLOGY.md, extract a component specification:

1. **HTML fragment** — the section's inner structure, cleaned of content-specific text
2. **CSS fragment** — all CSS rules that apply to this section (find matching `<style>` blocks or inline styles)
3. **Asset mapping** — which images/videos/fonts belong to this section
4. **Interaction notes** — what happens on scroll, hover, click

**Output: Write `$cloneDir/specs/{section-name}.md`** for each section:

```markdown
# Spec: {Section Name}

## Structure
```html
<!-- Cleaned HTML structure with content replaced by {{placeholders}} -->
```

## Styles
```css
/* CSS rules specific to this section */
```

## Assets
- `images/hero/bg.webp` → background image
- `fonts/Geist-Bold.woff2` → heading font

## Behavior
- Scroll-triggered fade-in (GSAP ScrollTrigger)
- Nav hover → dropdown menu with 300ms transition
- (list all observed interactions)
```

### Phase 5: Framework Generation

Generate an Astro project from the specs. The project structure:

```
brand-site/
├── src/
│   ├── config.ts              # Brand configuration (colors, content, meta)
│   ├── pages/
│   │   └── index.astro        # Composes all sections
│   ├── layouts/
│   │   └── BaseLayout.astro   # HTML shell (<head>, meta, GA, fonts)
│   ├── components/
│   │   ├── Nav.astro          # Navigation bar
│   │   ├── Hero.astro         # Hero section
│   │   ├── {Section}.astro    # One component per topology section
│   │   └── Footer.astro       # Site footer
│   └── styles/
│       └── global.css         # Tailwind imports + CSS custom properties
├── public/
│   ├── images/                # Copied from clone
│   ├── videos/                # Copied from clone
│   └── fonts/                 # Copied from clone
└── astro.config.ts
```

**Generation rules:**

1. **`config.ts` first** — all brand-specific values (colors, text, URLs) go here first, then components reference them:
   ```typescript
   export const BRAND = {
     name: "Brand Name",
     primaryColor: "oklch(0.205 0 0)",
     // ... all values from design-tokens.json
   };
   ```

2. **`BaseLayout.astro`** — copy the original HTML `<head>` structure, but replace hardcoded values with `{BRAND.*}` references. Move GA/GTM IDs to config.

3. **One component per section** — paste the cleaned HTML from each spec into its `.astro` file. Replace content text with `{BRAND.*}` or component props. Add the section's CSS as a `<style>` block inside the component.

4. **Asset paths** — copy from `$cloneDir` to `public/`. Change paths in components to `/images/...`, `/videos/...`, etc.

5. **Interaction re-creation** — for GSAP/Framer Motion animations, re-implement in the component's `<script>` block. Use the notes from Phase 4 as reference. Add `is:inline` if scripts share global scope.

6. **Verify with dev server:**
   ```bash
   cd brand-site && npx astro dev
   ```
   Open in Playwright, screenshot, compare against the clone screenshot from Phase 1.

### Phase 6: Validation Checklist

- [ ] `npx astro build` succeeds with no errors
- [ ] Page renders at correct dimensions (compare screenshots)
- [ ] All sections present and in correct order
- [ ] Navigation links work (even if pointing to placeholder URLs)
- [ ] Colors match design tokens (±5% perceptual difference is OK)
- [ ] Fonts load correctly
- [ ] Mobile responsive (check at 375px and 768px widths)
- [ ] No console errors on the generated site
- [ ] `config.ts` contains all customizable values (no hardcoded strings in components)

---

## Stop Conditions

| Status | Criteria |
|--------|----------|
| **DONE** | Astro build passes, visual match confirmed, config.ts has all brand values |
| **DONE_WITH_CONCERNS** | Build passes but some complex animations simplified or missing |
| **BLOCKED** | Clone directory not found (run /site-clone first) or empty |

---

## What This Skill Does NOT Do

- **Does NOT clone websites** — use `/site-clone` for that
- **Does NOT auto-guess brand content** — the user provides their own text, logo, colors
- **Does NOT guarantee pixel-perfect animation recreation** — complex GSAP/Three.js may need manual tuning
- **Does NOT handle e-commerce logic** — product pages, carts, checkout need custom backend

---

## After Completion

- `brand-site/` directory with working `npx astro dev`
- Tell user: "Edit `src/config.ts` to change brand name, colors, and content. Components are in `src/components/`."
- Leave the dev server running for inspection
