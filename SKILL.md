---
name: site-clone
description: |
  One-shot website cloning — navigate a URL, capture all assets, rewrite paths,
  and produce a fully offline copy. Iterates until zero console errors.
  Use when asked to "clone this site", "复刻这个网站", "save this page offline",
  "mirror this page".
triggers:
  - clone this site
  - 复刻网站
  - save this page offline
  - mirror this page
  - 扒站
  - archive website
allowed-tools:
  - Bash
  - PowerShell
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_network_requests
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_take_screenshot
  - mcp__playwright__browser_console_messages
  - mcp__bouncy__fetch
  - mcp__bouncy__scrape
  - mcp__bouncy__extract_text
  - mcp__bouncy__extract_title
---

# Site Clone — Production-Grade Website Mirroring Skill

One-shot website cloning workflow: Setup → Capture → Download → Rewrite → Validate → Fix → Done.

---

## Workflow

### Step 1: Setup Clone Directory

```powershell
$domain = "example.com"
$cloneDir = "E:\Projects\claude-code\site-clones\$domain"
New-Item -ItemType Directory -Path $cloneDir -Force | Out-Null
```

**Note:** Adjust the base path (`E:\Projects\claude-code\site-clones\`) to match the user's environment. This is the default; create wherever the user prefers.

---

### Step 2: Capture HTML

Two capture modes are available:

#### Mode A: Full (Playwright — Claude Code recommended)

**Step 2a — Navigate and wait:**
```
mcp__playwright__browser_navigate → url
```
Wait 3-5 seconds for JS execution. For infinite-scroll pages, also scroll to bottom:
```
mcp__playwright__browser_evaluate → () => { window.scrollTo(0, document.body.scrollHeight); return 'scrolled'; }
```
Then wait 2 more seconds.

**Step 2b — Capture rendered HTML including Shadow DOM:**
```javascript
// Serialize full DOM including shadow roots
() => {
  function serializeWithShadow(root) {
    let html = root.outerHTML || root.textContent || '';
    if (root.shadowRoot) {
      html += '<template shadowrootmode="open">' + serializeWithShadow(root.shadowRoot) + '</template>';
    }
    for (const child of root.children || []) {
      html += serializeWithShadow(child);
    }
    return html;
  }
  return serializeWithShadow(document.documentElement);
}
```
Use: `mcp__playwright__browser_evaluate` with the above function.

**Step 2c — Capture ALL resources the browser loaded (silver bullet):**
```javascript
// Returns every URL the browser fetched: CSS fonts, dynamic imports, workers, everything
() => JSON.stringify(performance.getEntriesByType('resource').map(e => ({
  url: e.name,
  type: e.initiatorType  // 'script', 'css', 'img', 'font', 'xmlhttprequest', 'other'
})))
```
Use: `mcp__playwright__browser_evaluate` with the above function. **This replaces regex-based asset discovery** — it catches `@font-face` fonts, CSS `@import`, Web Workers, dynamic `import()` chunks, and JS-constructed URLs that regex can never find.

#### Mode B: Static (curl — cross-tool compatible)

```powershell
curl.exe -s -L --max-time 30 `
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/131.0.0.0" `
  -o "$cloneDir\index.html" "https://$domain/"
```
**Limitations in Static mode:** No JS-rendered content, no Shadow DOM, no Performance API. Only assets discoverable via HTML attribute regex. Documented in Known Limitations below.

---

### Step 3: Save HTML — Correct UTF-8 Encoding (CRITICAL)

**The #1 source of failures is UTF-8 double-encoding.** PowerShell's `Set-Content` and `Out-File` default to UTF-16 LE on PowerShell 5.1, which corrupts multi-byte characters.

```powershell
# CORRECT — explicit UTF-8 without BOM
[System.IO.File]::WriteAllText(
    "$cloneDir\index.html",
    $htmlContent,
    [System.Text.UTF8Encoding]::new($false)
)
```

**If downloading from a URL directly:**
```powershell
$web = New-Object System.Net.WebClient
$web.Headers.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/131.0.0.0")
$web.Encoding = [System.Text.Encoding]::UTF8
$web.DownloadFile("https://$domain/", "$cloneDir\index.html")
```

---

### Step 4: Verify Encoding Integrity

Immediately after saving, spot-check non-ASCII characters:

```powershell
$html = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8
$title = [regex]::Match($html, '<title>(.+?)</title>').Groups[1].Value
# If the title looks garbled (e.g. "æ ¸é»" instead of CJK), encoding is corrupted.
# Re-download with [System.Net.WebClient] (Step 3 fallback).
Write-Host "Title: $title"
Write-Host "Size: $($html.Length) bytes"
```

Also verify `charset` meta tag is present and file extension associations are correct.

---

### Step 5: Extract & Download All Assets

**Primary — Use the Performance API resource list (Step 2c output) as ground truth:**

```powershell
# Parse the JSON from Step 2c: performance.getEntriesByType('resource')
# This list already includes: CSS @font-face fonts, dynamic imports, workers, source maps, etc.
$perfData = '[{"url":"https://domain.com/images/x.webp","type":"img"},...]' # paste from Step 2c
$resources = $perfData | ConvertFrom-Json

# Filter: same-origin only, skip the HTML page itself, skip data: URIs
$assets = $resources | Where-Object {
    $_.url -like "https://$domain/*" -and
    $_.url -notlike "https://$domain/" -and
    $_.url -notlike "https://$domain`?" -and
    $_.url -notlike "data:*"
} | ForEach-Object { $_.url -replace "https://$domain/", '' } | Sort-Object -Unique
```

**Secondary — Also regex-scan the HTML for assets NOT fetched by the browser (e.g. data-attributes on hidden elements):**

```powershell
$html = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8

# Comprehensive attribute pattern — includes track, embed, object, and all data-attrs
$pattern = '(?:src|href|content|poster|srcset|data-src|data-video|data-image|data-mobile-image|data-mobile-video|data-thumb|data-mobile-thumb|data-defer-src|data-nav-src|data)\s*=\s*["'']([^"'']+)["'']'

$htmlAssets = [regex]::Matches($html, $pattern) | ForEach-Object { $_.Groups[1].Value } | Where-Object {
    $_ -notmatch '^https?://' -and
    $_ -notmatch '^(#|data:|javascript:|mailto:|tel:)' -and
    $_ -match '\.(webp|png|jpg|jpeg|svg|gif|avif|ico|bmp|mp4|webm|mov|mp3|wav|woff2?|ttf|otf|eot|js|mjs|cjs|css|json|xml|csv|pdf|vtt|map|txt|zip)$'
} | ForEach-Object { $_ -replace '^\./', '' -replace '^/', '' } | Sort-Object -Unique

# Merge both lists
$assets = @($assets + $htmlAssets | Sort-Object -Unique)
Write-Host "Assets to download: $($assets.Count)"
```

**Step 5b — Check for `<base>` tag before downloading:**
The `<base href="...">` tag changes how relative URLs are resolved. Detect it:
```powershell
$baseTag = [regex]::Match($html, '<base\s+[^>]*href\s*=\s*["\']([^"\']+)').Groups[1].Value
if ($baseTag) { Write-Host "WARNING: <base> tag found: $baseTag — path resolution may be affected" }
```

**Step 5c — Create subdirectories and download:**

```powershell
$baseUrl = "https://$domain"
$failed = @()

foreach ($asset in $assets) {
    # Skip empty, paths with query-only, or already-downloaded
    if (-not $asset -or $asset -match '^[?#]') { continue }
    
    $localPath = Join-Path $cloneDir $asset
    $localDir = Split-Path $localPath -Parent
    if (-not (Test-Path $localDir)) { New-Item -ItemType Directory -Path $localDir -Force | Out-Null }

    try {
        curl.exe -s -L --max-time 30 -o $localPath "$baseUrl/$asset" 2>&1 | Out-Null
        if ((Test-Path $localPath) -and ((Get-Item $localPath).Length -gt 100)) {
            Write-Host "  OK: $asset"
        } else {
            Remove-Item $localPath -Force -ErrorAction SilentlyContinue
            Write-Host "  EMPTY/TINY: $asset"
            $failed += $asset
        }
    } catch {
        Write-Host "  FAIL: $asset"
        $failed += $asset
    }
}

Write-Host "Downloaded: $($assets.Count - $failed.Count) / $($assets.Count)"
if ($failed.Count -gt 0) { Write-Host "Failed: $($failed.Count) — will retry in validation pass" }
```

---

### Step 6: Rewrite Paths

Only rewrite the target domain's absolute paths to relative. Never touch external CDN URLs.

```powershell
$html = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8

# Replace absolute domain paths with relative (systematic — not per-directory)
$escapedDomain = [regex]::Escape($domain)
$html = $html -replace "https?://${escapedDomain}/(images/|videos/|fonts/|js/|css/|assets/|_nuxt/|static/)", './$1'
$html = $html -replace 'src="/_nuxt/', 'src="./_nuxt/'
$html = $html -replace 'href="/_nuxt/', 'href="./_nuxt/'

# Save with correct encoding
[System.IO.File]::WriteAllText("$cloneDir\index.html", $html, [System.Text.UTF8Encoding]::new($false))

Write-Host "Paths rewritten"
```

**Rule:** If the original HTML uses relative paths (like `images/x.webp` without leading `/`), no rewriting is needed.

---

### Step 7: Deep Comparison — Byte-Level Verification

Compare the local (rewritten) HTML against the live HTML to catch encoding corruption, missing elements, or truncation:

```powershell
$live = Get-Content "$env:TEMP\live_compare.html" -Raw -Encoding UTF8
$local = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8

# Normalize the live HTML with the same rewrites applied to local
$liveNormalized = $live
$escapedDomain = [regex]::Escape($domain)
$liveNormalized = $liveNormalized -replace "https?://${escapedDomain}/(images/|videos/|fonts/|js/|css/|assets/|_nuxt/|static/)", './$1'
$liveNormalized = $liveNormalized -replace 'src="/_nuxt/', 'src="./_nuxt/'
$liveNormalized = $liveNormalized -replace 'href="/_nuxt/', 'href="./_nuxt/'

if ($liveNormalized -eq $local) {
    Write-Host "PERFECT MATCH — local HTML is byte-identical to live (after path normalization)"
} else {
    # Report size difference
    Write-Host "MISMATCH: live=$($liveNormalized.Length) local=$($local.Length)"
}
```

---

### Step 8: Start Local Server

Save `server.js` to the clone directory:

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');
const baseDir = __dirname;
const mime = {
  '.html': 'text/html; charset=utf-8', '.css': 'text/css', '.js': 'application/javascript',
  '.svg': 'image/svg+xml', '.webp': 'image/webp', '.woff': 'font/woff',
  '.woff2': 'font/woff2', '.png': 'image/png', '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg', '.ico': 'image/x-icon', '.mp4': 'video/mp4',
  '.json': 'application/json', '.gif': 'image/gif', '.avif': 'image/avif',
  '.webm': 'video/webm', '.ttf': 'font/ttf', '.otf': 'font/otf',
  '.xml': 'application/xml', '.pdf': 'application/pdf', '.txt': 'text/plain'
};

http.createServer((req, res) => {
  let file = req.url === '/' ? '/index.html' : req.url.split('?')[0];
  file = decodeURIComponent(file).replace(/\.\.\//g, '').replace(/\\/g, '/');
  const filePath = path.join(baseDir, file);
  fs.readFile(filePath, (err, data) => {
    if (err) { res.writeHead(404, { 'Content-Type': 'text/plain' }); res.end('Not Found'); return; }
    const ext = path.extname(file).toLowerCase();
    res.writeHead(200, { 'Content-Type': mime[ext] || 'application/octet-stream' });
    res.end(data);
  });
}).listen(8765, () => console.log('http://localhost:8765/'));
```

**Start server robustly:**

```powershell
# Kill any previous process on port 8765
$existing = netstat -ano | Select-String ":8765.*LISTENING"
if ($existing) {
    $procId = $existing -replace ".*LISTENING\s+(\d+).*", '$1'
    taskkill /PID $procId /F 2>&1 | Out-Null
    Start-Sleep 1
}

Start-Process node -ArgumentList "server.js" -WorkingDirectory $cloneDir -WindowStyle Hidden
Start-Sleep 2

# Verify
try {
    $resp = Invoke-WebRequest -Uri "http://localhost:8765/" -TimeoutSec 5 -UseBasicParsing
    if ($resp.StatusCode -eq 200) { Write-Host "Server OK: http://localhost:8765/" }
} catch {
    Write-Host "ERROR: Server failed to start — check Node.js is installed"
}
```

---

### Step 9: Validate — Zero Console Error Loop

1. Navigate Playwright to `http://localhost:8765/`:
   ```
   mcp__playwright__browser_navigate → http://localhost:8765/
   ```
2. Read console errors:
   ```
   mcp__playwright__browser_console_messages → level: error
   ```
3. Extract 404 URLs from error messages → download each from `https://$domain/...`
4. Reload and repeat until **console errors = 0**
5. Acceptable warnings (ignore): "Slow network" font loading, CORS for external CDNs
6. If Playwright is unavailable: curl spot-check 10-15 key asset URLs against localhost

---

### Step 10: Generate Manifest

```powershell
$files = Get-ChildItem -Path $cloneDir -Recurse -File | Where-Object { $_.Name -notin @("server.js", "site-manifest.json") }
$totalSize = ($files | Measure-Object Length -Sum).Sum

$manifest = @{
    source = "https://$domain/"
    cloned_at = (Get-Date -Format "yyyy-MM-ddTHH:mm:sszzz")
    total_files = $files.Count
    total_size_mb = [math]::Round($totalSize / 1MB, 1)
    pages = @("/")
    assets = @{
        images = ($files | Where-Object { $_.Extension -match '\.(webp|png|jpg|jpeg|svg|gif|avif|ico)$' }).Count
        videos = ($files | Where-Object { $_.Extension -match '\.(mp4|webm)$' }).Count
        scripts = ($files | Where-Object { $_.Extension -eq '.js' }).Count
        styles = ($files | Where-Object { $_.Extension -eq '.css' }).Count
        fonts = ($files | Where-Object { $_.Extension -match '\.(woff2?|ttf|otf|eot)$' }).Count
        html = ($files | Where-Object { $_.Extension -eq '.html' }).Count
    }
    missing = @()
    local_url = "http://localhost:8765/"
}

$manifest | ConvertTo-Json -Depth 3 |
    Out-File -FilePath "$cloneDir\site-manifest.json" -Encoding utf8
```

---

### Step 11: Screenshot Comparison

Take a full-page screenshot of `http://localhost:8765/` and compare visually with the live site.

---

## Common Pitfalls & Fixes

| # | Pitfall | Symptom | Root Cause | Fix |
|---|---------|---------|------------|-----|
| 1 | **UTF-8 double-encoding** | CJK/emoji show as `æ ¸é»` | `Set-Content`/`Out-File` defaults to UTF-16 LE | Use `[System.IO.File]::WriteAllText` with `UTF8Encoding($false)` |
| 2 | **Missing lazy-loaded assets** | Console 404s for images/videos hidden in data-attributes | `data-src`/`data-image`/`data-defer-src` not in download list | Use the exhaustive grep pattern in Step 5a |
| 3 | **Server port conflict** | `EADDRINUSE` error | Previous node process still on port 8765 | Kill existing process before starting (Step 8) |
| 4 | **Playwright disconnects** | Tool not available mid-session | MCP server dropped | Use bouncy/curl for capture; curl for validation |
| 5 | **Tiny/broken downloads** | Files < 100 bytes | CDN returned error page instead of asset | Post-download size check: delete if < 100 bytes |
| 6 | **Root-level assets missed** | `transitions.js`, `noise.js` not downloaded | Asset pattern only matches subdirectories | Step 5a pattern uses `[^"]+` not `(images/|...)` |
| 7 | **CSS url() not captured** | Background images 404 | `url()` references in inline CSS not matched by attr grep | Add: `$html -match 'url\(([^)]+)\)'` for inline styles |
| 8 | **HTML charset not set** | Browser shows garbled text | Server sends `Content-Type: text/html` without charset | Use `text/html; charset=utf-8` in server MIME config |

---

## Asset Discovery — Complete Pattern Reference

Use these patterns to find every asset in the HTML:

**Pattern 1 — HTML attributes (standard + data-attributes + media elements):**
```
(?:src|href|content|poster|srcset|data|data-src|data-video|data-image|data-mobile-image|data-mobile-video|data-thumb|data-mobile-thumb|data-defer-src|data-nav-src)\s*=\s*["']([^"']+)["']
```
This also catches `<track src>`, `<embed src>`, `<object data>`, and `<source src>` / `<source srcset>`.

**Pattern 2 — CSS url() in inline styles:**
```
url\(\s*["']?([^"')]+)["']?\s*\)
```

**Pattern 3 — srcset (responsive images):**
```
srcset\s*=\s*["']([^"']+)["']
```
Then split each match by `,` and take the path from each `url size` pair.

**Filtering rules for all patterns:**
- Exclude: `https?://` (external CDN — do not download)
- Exclude: `data:` (inline base64 — already embedded)
- Exclude: `#` (anchor links)
- Exclude: `javascript:` (JS pseudo-links)
- Exclude: `mailto:` / `tel:` (contact links)
- Include only: paths with a known static file extension

**Known extensions:**
```
\.(webp|png|jpg|jpeg|svg|gif|avif|ico|bmp|
   mp4|webm|mov|mp3|wav|ogg|
   woff2?|ttf|otf|eot|
   js|mjs|cjs|css|json|xml|csv|pdf|zip)
```

---

## Known Limitations — What the Skill Cannot Clone

These are fundamental constraints of snapshot-based static cloning, not bugs.

| Limitation | Why | Workaround |
|------------|-----|------------|
| **WebSocket / SSE / GraphQL subscriptions** | Live streaming protocols; no static file to capture | Will fail silently in offline copy. Not fixable. |
| **IndexedDB / localStorage state** | Client-side database; page may render differently without it | Not fixable generically. Site-specific. |
| **Service Worker cache & logic** | SW lifecycle tied to origin; can't register on `localhost` copies | Not fixable. Outside single-page scope. |
| **Canvas-generated images** | Procedurally rendered at runtime via WebGL/2D API | Per-site custom `canvas.toDataURL()` call. |
| **Cross-origin iframes** | Browser security prevents content access | Unavoidable without a proxy server. |
| **Auth-gated content** | Login wall / paywall / member-only sections | Documented as **BLOCKED** stop condition. |
| **A/B test variants** | Only one variant captured per snapshot | Re-run to capture other variants. |
| **Geo-IP targeted content** | Capture server location determines variant | Use VPN when Playwright/bouncy has proxy support. |
| **Infinite scroll (without scroll step)** | Only initial items in DOM | Use the scroll-to-bottom step in Step 2a. |
| **`eval()` / obfuscated JS asset URLs** | URLs constructed at runtime inside JS closures | Performance API catches these (Step 2c). |
| **Multi-page site sub-pages** | Skill clones ONE page deeply, not a crawl | Run the skill again for each sub-page URL. |
| **sitemap.xml / robots.txt** | Not linked from page HTML | Manually `curl https://domain.com/robots.txt`. |

---

## Cross-Tool Compatibility

| AI Tool | Capture | Asset Discovery | Server | Best Mode |
|---------|---------|-----------------|--------|-----------|
| **Claude Code** | Playwright + bouncy | Performance API + regex | Node.js | **Full** — all features |
| **Codex CLI** | curl only | Regex only | Node.js (if available) | **Static** — no JS rendering |
| **Cursor** | curl only | Regex only | Node.js (if available) | **Static** — no JS rendering |
| **GitHub Copilot** | Manual (user copies HTML) | Regex only | Node.js | **Static / Manual** |
| **aider / terminal AI** | curl only | Regex only | `python3 -m http.server` fallback | **Static** |
| **ChatGPT (Code Interpreter)** | Not supported | N/A | Not supported | **Not compatible** |

**Minimum requirements:**
- **Static mode:** curl, Node.js (or Python 3 for `http.server`), file write capability
- **Full mode:** All of the above + Playwright MCP + Performance API access

**For non-Claude-Code users:** The Static mode produces a functional offline copy, but JS-rendered content, CSS-loaded fonts, dynamic imports, and Shadow DOM will be missing. Document this clearly when someone installs the skill on a different tool.

---

## Stop Conditions

| Status | Criteria |
|--------|----------|
| **DONE** | 0 console errors AND HTML byte-exact match AND visual match confirmed |
| **DONE_WITH_CONCERNS** | ≤ 2 minor missing assets that don't affect layout (e.g. favicon) |
| **BLOCKED** | Original site requires login / CAPTCHA / bot detection that can't be bypassed |

---

## Output Checklist

After completion, report to the user:
- [ ] Local URL (`http://localhost:8765/`)
- [ ] Clone directory path
- [ ] File count and total size
- [ ] HTML exact match status (byte-level)
- [ ] Any assets that couldn't be downloaded and why

---

## Cleanup

Do NOT delete the clone directory or stop the server — leave them running for the user to inspect.
