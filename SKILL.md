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
  - mcp__playwright__browser_click
  - mcp__playwright__browser_hover
  - mcp__bouncy__fetch
  - mcp__bouncy__scrape
  - mcp__bouncy__extract_text
  - mcp__bouncy__extract_title
---

# Site Clone — Production-Grade Website Mirroring Skill

One-shot website cloning workflow: Pre-flight → Capture (interaction sweep) → Download → Rewrite (dynamic) → Validate → Sitemap → Done.

---

## Workflow

### Step 0: Pre-Flight Checklist

Before starting the clone, run these checks to fail fast on unsupported targets:

```powershell
$url = "https://$domain/"
Write-Host "=== Pre-flight: $domain ==="

# 1. Accessibility check
try {
    $resp = Invoke-WebRequest -Uri $url -TimeoutSec 10 -UseBasicParsing -Method Head
    Write-Host "  HTTP: $($resp.StatusCode)"
} catch {
    Write-Host "  BLOCKED: Site unreachable — $($_.Exception.Message)"
    # STOP here — site is down or behind a wall
}

# 2. Framework fingerprint (informs asset path patterns)
$html = curl.exe -s -L --max-time 15 "$url"
switch -Regex ($html) {
    '__NEXT_DATA__|_next/'  { Write-Host "  FRAMEWORK: Next.js" }
    '__NUXT__|_nuxt/'       { Write-Host "  FRAMEWORK: Nuxt" }
    'wp-content|wp-includes' { Write-Host "  FRAMEWORK: WordPress" }
    'shopify'               { Write-Host "  FRAMEWORK: Shopify" }
    'data-reactroot'        { Write-Host "  FRAMEWORK: React (CRA)" }
    '<astro-'               { Write-Host "  FRAMEWORK: Astro" }
}

# 3. Bot detection check
if ($html -match 'cf-turnstile|_cf_chl_opt|recaptcha|hcaptcha|challenge-platform') {
    Write-Host "  WARNING: CAPTCHA/bot detection found — Full mode may be blocked"
}

# 4. Auth gate check
if ($html -match 'login|signin|password' -and $html -match '<form') {
    Write-Host "  WARNING: Login form detected — may be auth-gated"
}
```

**Stop conditions at Step 0:**
- Site returns 403/503 → **BLOCKED** (try again later)
- CAPTCHA wall → **BLOCKED** (document in output)
- Auth redirect → **BLOCKED** (needs credentials)

---

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

**Step 2a — Navigate, then interaction sweep (scroll → hover → click):**

Navigate to the target URL:
```
mcp__playwright__browser_navigate → url
```
Wait 3 seconds for initial JS execution.

**Interaction sweep — capture all dynamic states before saving HTML:**

```
1. SCROLL to trigger lazy-loaded content and scroll-based animations:
   mcp__playwright__browser_evaluate → 
   () => {
     const height = document.body.scrollHeight;
     const steps = 10;
     for (let i = 1; i <= steps; i++) {
       window.scrollTo(0, (height / steps) * i);
     }
     window.scrollTo(0, 0);  // back to top
     return `scrolled ${height}px in ${steps} steps`;
   }
   Wait 2 seconds after scroll.

2. HOVER over navigation items to reveal dropdowns:
   FIRST — take a snapshot to identify hover targets:
   mcp__playwright__browser_snapshot →
   (Find interactive elements matching nav links, menu items, dropdown triggers.
    Note their `index` values from the snapshot's interactive list.)
   
   For each matched element:
   mcp__playwright__browser_hover → index (from snapshot)
   Wait 300ms between each hover.
   mcp__playwright__browser_snapshot → capture revealed dropdown content

3. CLICK through carousels/tabs to capture all slides/panels:
   FIRST — detect controls via evaluate:
   mcp__playwright__browser_evaluate →
   () => {
     const controls = [
       ...document.querySelectorAll('[class*=carousel] button, [class*=slider] button, [class*=tab], .swiper-button, [role=tab]')
     ];
     return JSON.stringify(controls.map(c => ({ text: c.textContent?.trim().slice(0,40), tag: c.tagName, className: c.className?.slice(0,60) })));
   }
   
   THEN — take a snapshot to get Playwright's interactive element indices:
   mcp__playwright__browser_snapshot →
   (Match the detected controls from the evaluate output to snapshot elements by text/tag/class.
    Use the `index` from the snapshot's interactive list — NOT the JSON array position.)
   
   For each matched control:
   mcp__playwright__browser_click → index (from snapshot)
   Wait 500ms after each click for animations/content swap
   mcp__playwright__browser_evaluate → () => document.documentElement.outerHTML
   (Save each state's HTML as a variant in $cloneDir/variants/)

4. RETURN to default state:
   mcp__playwright__browser_navigate → url (reload)
   Wait 2 seconds.
```

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
    $_.url -notlike "https://$domain[?]*" -and
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

# --- Also extract url() references from inline <style> blocks and style="" attributes ---
$urlPattern = 'url\(\s*["'']?([^"'')\s]+)["'']?\s*\)'
$cssAssets = [regex]::Matches($html, $urlPattern) | ForEach-Object { $_.Groups[1].Value } | Where-Object {
    $_ -notmatch '^https?://' -and
    $_ -notmatch '^(#|data:|javascript:)'
} | ForEach-Object { $_ -replace '^\./', '' -replace '^/', '' -replace '\?.*$', '' } | Sort-Object -Unique

# Merge all three lists
$assets = @($assets + $htmlAssets + $cssAssets | Sort-Object -Unique)
Write-Host "Assets to download: $($assets.Count) (Performance API + $($htmlAssets.Count) attrs + $($cssAssets.Count) CSS urls)"
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
        curl.exe -s -L --max-time 30 -o $localPath "$baseUrl/$asset"
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

# --- Step 5d: Scan downloaded CSS files for url() references ---
# External CSS files may reference fonts, background images, etc. via url()
# that neither Performance API nor HTML attribute scanning caught.
Write-Host "`nScanning CSS files for url() references..."
$cssFiles = Get-ChildItem -Path $cloneDir -Recurse -Filter "*.css" -File
$cssDiscovered = @()

foreach ($cssFile in $cssFiles) {
    $cssContent = Get-Content $cssFile.FullName -Raw -Encoding UTF8
    $urls = [regex]::Matches($cssContent, 'url\(\s*["'']?([^"'')\s]+)["'']?\s*\)') |
        ForEach-Object { $_.Groups[1].Value } |
        Where-Object {
            $_ -notmatch '^https?://' -and
            $_ -notmatch '^(#|data:)' -and
            $_ -match '\.(woff2?|ttf|otf|eot|webp|png|jpg|jpeg|svg|gif|avif|ico|mp4|webm)$'
        } |
        ForEach-Object { $_ -replace '^\./', '' -replace '^/', '' -replace '\?.*$', '' }

    foreach ($url in $urls) {
        # CSS url() paths are relative to the CSS file's directory
        $cssDir = Split-Path $cssFile.FullName -Parent
        $resolvedPath = Join-Path $cssDir $url
        $relativePath = $resolvedPath -replace [regex]::Escape($cloneDir), '' -replace '^\\', '' -replace '\\', '/'
        
        if (-not (Test-Path $resolvedPath) -and $relativePath -notin $assets) {
            $cssDiscovered += $relativePath
        }
    }
}

$cssDiscovered = $cssDiscovered | Sort-Object -Unique
if ($cssDiscovered.Count -gt 0) {
    Write-Host "  Found $($cssDiscovered.Count) additional CSS assets (fonts, bg images)"
    foreach ($asset in $cssDiscovered) {
        $localPath = Join-Path $cloneDir $asset
        $localDir = Split-Path $localPath -Parent
        if (-not (Test-Path $localDir)) { New-Item -ItemType Directory -Path $localDir -Force | Out-Null }
        try {
            curl.exe -s -L --max-time 30 -o $localPath "$baseUrl/$asset"
            if ((Test-Path $localPath) -and ((Get-Item $localPath).Length -gt 100)) {
                Write-Host "    OK: $asset"
                $assets += $asset
            } else {
                Remove-Item $localPath -Force -ErrorAction SilentlyContinue
                Write-Host "    EMPTY: $asset"
            }
        } catch { Write-Host "    FAIL: $asset" }
    }
} else {
    Write-Host "  No additional CSS assets found"
}
```

---

### Step 6: Rewrite Paths — Dynamic Prefix Discovery

**Do NOT hardcode directory names.** Instead, auto-discover all unique path prefixes from the actual downloaded asset list, then generate rewrite rules dynamically.

```powershell
$html = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8
$escapedDomain = [regex]::Escape($domain)

# 6a — Discover all unique directory prefixes from downloaded assets
$prefixes = $assets | ForEach-Object {
    if ($_ -match '^([a-zA-Z0-9_-]+)/') { $matches[1] }
    elseif ($_ -match '^([a-zA-Z0-9_-]+)$') { $null }  # root-level file, skip
} | Where-Object { $_ } | Sort-Object -Unique

Write-Host "Discovered path prefixes: $($prefixes -join ', ')"

# 6b — Generate and apply rewrite rules for each discovered prefix
foreach ($prefix in $prefixes) {
    $escapedPrefix = [regex]::Escape($prefix)
    
    # Absolute URLs → relative
    $html = $html -replace "https?://${escapedDomain}/${escapedPrefix}/", "./${prefix}/"
    $html = $html -replace "https?://${escapedDomain}/${escapedPrefix}`"", "./${prefix}`""
    
    # Root-relative paths → relative  
    $html = $html -replace "src=`"/${escapedPrefix}/", "src=`"./${prefix}/"
    $html = $html -replace "href=`"/${escapedPrefix}/", "href=`"./${prefix}/"
    $html = $html -replace "url\(`"/${escapedPrefix}/", "url(`"./${prefix}/"
    $html = $html -replace "url\(/${escapedPrefix}/", "url(./${prefix}/"
    
    # Single-quoted variants
    $html = $html -replace "src='/${escapedPrefix}/", "src='./${prefix}/"
    $html = $html -replace "href='/${escapedPrefix}/", "href='./${prefix}/"
}

# 6c — Also rewrite root-level asset files (e.g. /transitions.js, /noise.js)
$rootAssets = $assets | Where-Object { $_ -notmatch '/' } | ForEach-Object { $_ -replace '\?.*$', '' } | Sort-Object -Unique
foreach ($asset in $rootAssets) {
    $escapedAsset = [regex]::Escape($asset)
    $html = $html -replace "https?://${escapedDomain}/${escapedAsset}", "./${asset}"
    $html = $html -replace "src=`"/${escapedAsset}`"", "src=`"./${asset}`""
    $html = $html -replace "href=`"/${escapedAsset}`"", "href=`"./${asset}`""
    $html = $html -replace "src='/${escapedAsset}'", "src='./${asset}'"
    $html = $html -replace "href='/${escapedAsset}'", "href='./${asset}'"
}

# 6d — Insert <base href="./"> as a safety net for any missed paths
if ($html -notmatch '<base\s+[^>]*href') {
    $html = $html -replace '(<head[^>]*>)', "`$1`n  <base href=`"./`">"
    Write-Host "  Inserted <base href=`"./`">"
} else {
    Write-Host "  Existing <base> tag preserved"
}

# 6e — Save with correct encoding
[System.IO.File]::WriteAllText("$cloneDir\index.html", $html, [System.Text.UTF8Encoding]::new($false))

Write-Host "Paths rewritten: $($prefixes.Count) prefixes + $($rootAssets.Count) root assets"
```

**Cleanup — remove known CDN artifacts that produce spurious 404s:**

```powershell
# Cloudflare-specific artifacts that need removal (not download):
$html = $html -replace '<script[^>]*cdn-cgi/scripts/[^<]*email-decode[^<]*</script>', ''
$html = $html -replace '<script[^>]*cloudflare[^<]*(rocket-loader|email-protection|cf-beacon)[^<]*</script>', ''
$html = $html -replace '<span[^>]*__cf_email__[^>]*>.*?</span>', '<!-- cf email removed -->'

# Replace Cloudflare email obfuscation with direct mailto:
$html = $html -replace '\[email&#160;protected\]', 'contact@' + $domain

[System.IO.File]::WriteAllText("$cloneDir\index.html", $html, [System.Text.UTF8Encoding]::new($false))
```

**Rule:** Only rewrite paths belonging to the target domain. Never touch external CDN URLs (cdn.jsdelivr.net, unpkg.com, fonts.googleapis.com, etc.).

---

### Step 7: Deep Comparison — Byte-Level Verification

Compare the local (rewritten) HTML against the live HTML to catch encoding corruption, missing elements, or truncation:

```powershell
$live = Get-Content "$env:TEMP\live_compare.html" -Raw -Encoding UTF8
$local = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8

# Apply the SAME dynamic rewrites to the live HTML for fair comparison
$liveNormalized = $live
$escapedDomain = [regex]::Escape($domain)

foreach ($prefix in $prefixes) {
    $escapedPrefix = [regex]::Escape($prefix)
    $liveNormalized = $liveNormalized -replace "https?://${escapedDomain}/${escapedPrefix}/", "./${prefix}/"
    $liveNormalized = $liveNormalized -replace "src=`"/${escapedPrefix}/", "src=`"./${prefix}/"
    $liveNormalized = $liveNormalized -replace "href=`"/${escapedPrefix}/", "href=`"./${prefix}/"
    $liveNormalized = $liveNormalized -replace "src='/${escapedPrefix}/", "src='./${prefix}/"
    $liveNormalized = $liveNormalized -replace "href='/${escapedPrefix}/", "href='./${prefix}/"
}

foreach ($asset in $rootAssets) {
    $escapedAsset = [regex]::Escape($asset)
    $liveNormalized = $liveNormalized -replace "https?://${escapedDomain}/${escapedAsset}", "./${asset}"
    $liveNormalized = $liveNormalized -replace "src=`"/${escapedAsset}`"", "src=`"./${asset}`""
    $liveNormalized = $liveNormalized -replace "href=`"/${escapedAsset}`"", "href=`"./${asset}`""
}

if ($liveNormalized -eq $local) {
    Write-Host "PERFECT MATCH — local HTML is byte-identical to live (after path normalization)"
} else {
    # Show first differing line for debugging
    Write-Host "MISMATCH: live=$($liveNormalized.Length) local=$($local.Length)"
    for ($i = 0; $i -lt [Math]::Min($liveNormalized.Length, $local.Length); $i++) {
        if ($liveNormalized[$i] -ne $local[$i]) {
            $ctx = [Math]::Max(0, $i - 20)
            Write-Host "First diff at byte $i : context='$($liveNormalized.Substring($ctx, [Math]::Min(60, $liveNormalized.Length - $ctx)))'"
            break
        }
    }
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

### Step 12: Sitemap Discovery (Optional — Multi-Page)

For sites with multiple pages, extract the link structure to enable crawling:

**12a — Extract internal links from the cloned HTML:**

```powershell
$html = Get-Content "$cloneDir\index.html" -Raw -Encoding UTF8

# Extract all <a href> pointing to same domain
$internalLinks = [regex]::Matches($html, 'href="(https?://' + [regex]::Escape($domain) + '(/[^"#]*))"') |
    ForEach-Object { $_.Groups[2].Value } |
    Where-Object { $_ -notmatch '\.(webp|png|jpg|jpeg|svg|gif|ico|mp4|webm|pdf|zip)$' } |
    Sort-Object -Unique

Write-Host "Internal pages found: $($internalLinks.Count)"
$internalLinks | ForEach-Object { Write-Host "  $_" }
```

**12b — Also try common sitemap locations:**

```powershell
$sitemapUrls = @(
    "https://$domain/sitemap.xml",
    "https://$domain/sitemap_index.xml",
    "https://$domain/sitemap-index.xml",
    "https://$domain/robots.txt"
)
foreach ($url in $sitemapUrls) {
    try {
        $content = curl.exe -s -L --max-time 10 "$url"
        if ($content -match '<urlset|<sitemapindex|<url>|Sitemap:') {
            Write-Host "  FOUND: $url — save for crawling"
            $fileName = Split-Path $url -Leaf
            [System.IO.File]::WriteAllText("$cloneDir\$fileName", $content, [System.Text.UTF8Encoding]::new($false))
        }
    } catch { }
}
```

**12c — Update manifest with discovered pages:**

```powershell
$manifest = Get-Content "$cloneDir\site-manifest.json" -Raw | ConvertFrom-Json
$manifest.pages = @("/") + @($internalLinks | ForEach-Object { $_ })
$manifest | ConvertTo-Json -Depth 3 | Out-File -FilePath "$cloneDir\site-manifest.json" -Encoding utf8
```

**Note:** This step does NOT clone sub-pages automatically. Run the skill again for each discovered URL to clone them individually. For full-site crawling, consider `wget --mirror` as a complementary tool.

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
| 7 | **CSS url() not captured** | Background images 404 | `url()` references in CSS not matched by html attr grep | **FIXED v1.2:** Step 5a scans inline `<style>` + style="". Step 5d scans all downloaded `.css` files for url() and downloads missing assets |
| 8 | **HTML charset not set** | Browser shows garbled text | Server sends `Content-Type: text/html` without charset | Use `text/html; charset=utf-8` in server MIME config |
| 9 | **Cloudflare email obfuscation** | Emails show as `[email protected]` | CF's `email-decode.min.js` + `__cf_email__` spans | Remove both: strip CF email spans, use direct `mailto:` links |
| 10 | **Cloudflare analytics beacon** | 404 on `/cdn-cgi/rum` | CF Rocket Loader injects `cf-beacon` script | Remove `<script defer src='/cdn-cgi/...'>` blocks from HTML |
| 11 | **Nuxt `_nuxt/` paths** | CSS/JS chunks 404 | Nuxt uses hashed filenames under `_nuxt/` | `_nuxt/` prefix must be in the download list; hash filenames auto-discovered by Performance API |
| 12 | **Next.js `_next/` paths** | Static assets 404 | Next.js places hashed assets under `_next/static/` | Same as Nuxt — Performance API catches these; dynamic prefix discovery handles rewrites |
| 13 | **Astro hoisted scripts** | JS breaks in cloned copy | Astro hoists `<script>` to `<head>` as `type="module"` — scope isolation | If rebuilding in Astro: add `is:inline` directive to inline scripts. For raw clone: scripts already in original positions |
| 14 | **GSAP / animation libraries** | Animations don't run locally | ScrollTrigger tied to `window` dimensions that differ in headless capture | Not fixable — animations depend on runtime viewport. Static snapshot captures the default state |
| 15 | **WebGL / Three.js canvas** | 3D elements render as blank or frozen | Canvas content is procedural; no static file to capture | Use `canvas.toDataURL()` to snapshot the current frame during capture (Step 2a) |

---

## Framework-Specific Patterns

When the pre-flight (Step 0) detects a known framework, apply these targeted rules:

### Nuxt.js / Nuxt 3
- Asset path prefix: `_nuxt/` (auto-discovered by Performance API)
- Lazy routes: `_nuxt/builds/...` chunks loaded on scroll — interaction sweep triggers them
- Watch for: `__NUXT__` JSON payload in HTML — leave as-is (needed for hydration)

### Next.js
- Asset path prefix: `_next/static/`
- Watch for: `__NEXT_DATA__` JSON with page props — leave as-is
- Image optimization: `_next/image?url=...` URLs → save as regular images, rewrite to local paths

### WordPress
- Uploads: `wp-content/uploads/` (year/month structure)
- Theme assets: `wp-content/themes/{theme}/`
- Plugins: `wp-content/plugins/`
- Watch for: `?ver=` query strings on CSS/JS (cache busters) — strip during download

### Shopify
- CDN assets: usually on `cdn.shopify.com` (external — do not download)
- Product images: `//cdn.shopify.com/...` → try downloading if same-origin
- Theme assets: often on a separate `//theme-assets.` subdomain

### Astro
- Build output: `_astro/` prefix for hashed assets
- Inline scripts: preserved in original positions by `is:inline` (rebuild concern only)
- CSS: bundled into single hashed file, sometimes inlined as `<style>` in production

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
| **Multi-page site sub-pages** | Skill clones ONE page deeply, not a crawl | Step 12 discovers internal links + sitemaps. Run the skill again per URL, or use `wget --mirror` for full sites. |
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
