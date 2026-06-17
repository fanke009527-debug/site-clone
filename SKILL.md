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
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_network_requests
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_take_screenshot
  - mcp__playwright__browser_console_messages
---

# Site Clone — Optimized Website Mirroring Skill

One-shot website cloning workflow. Navigate → capture → download → rewrite → validate → fix → done.

## Workflow

### Step 1: Capture (one pass)

1. Navigate to the target URL with Playwright:
   ```
   mcp__playwright__browser_navigate → url
   ```
2. Wait 3 seconds for all lazy-loaded assets and route chunks to fire.
3. Capture ALL network requests (including static resources):
   ```
   mcp__playwright__browser_network_requests → static: true
   ```
4. Save the rendered HTML:
   ```
   mcp__playwright__browser_evaluate → () => document.documentElement.outerHTML
   ```

### Step 2: Setup clone directory

Create the clone directory at `E:\Projects\claude-code\site-clones\{domain}\` with subdirectories:
- Mirror the URL path structure from the network requests
- Extract unique directory paths from all asset URLs
- Create all needed directories in one PowerShell call

### Step 3: Download all assets

From the captured network requests list, download every 200-response asset.
**Do NOT manually enumerate — use the network log as source of truth.**

PowerShell pattern:
```powershell
$assets = @(<extract from network requests>)
foreach ($asset in $assets) {
    Invoke-WebRequest -Uri "$baseUrl$asset" -OutFile "$cloneDir\$asset"
}
```

### Step 4: Rewrite paths

Rewrite HTML to use local relative paths. Be surgical — only replace the target domain's absolute paths:

```powershell
$html = $html -replace 'href="/_nuxt/', 'href="./_nuxt/'
$html = $html -replace 'src="/_nuxt/', 'src="./_nuxt/'
$html = $html -replace 'href="/images/', 'href="./images/'
$html = $html -replace 'src="/images/', 'src="./images/'
$html = $html -replace 'href="/fonts/', 'href="./fonts/'
$html = $html -replace 'src="/fonts/', 'src="./fonts/'
$html = $html -replace 'url\(/images/', 'url(./images/'
$html = $html -replace 'url\(/fonts/', 'url(./fonts/'
```

**Rule:** Only rewrite paths belonging to the target domain. Never touch external CDN URLs (they stay as-is).

Also add a `<base href="./">` tag in `<head>` as a safety net.

### Step 5: Start local server

Use Node.js built-in `http` module (no dependencies):

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');
const baseDir = __dirname;
const mime = { '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript', '.svg': 'image/svg+xml', '.webp': 'image/webp', '.woff': 'font/woff', '.woff2': 'font/woff2', '.png': 'image/png', '.jpg': 'image/jpeg', '.ico': 'image/x-icon', '.mp4': 'video/mp4', '.json': 'application/json' };
http.createServer((req, res) => {
  let file = req.url === '/' ? '/index.html' : req.url.split('?')[0];
  fs.readFile(path.join(baseDir, decodeURIComponent(file)), (err, data) => {
    if (err) { res.writeHead(404); res.end('Not Found'); return; }
    res.writeHead(200, { 'Content-Type': mime[path.extname(file)] || 'application/octet-stream' });
    res.end(data);
  });
}).listen(8765, () => console.log('http://localhost:8765/'));
```

Save as `server.js` in the clone directory. Start with `node server.js &`.

### Step 6: Validate — Console Zero Error Loop

1. Navigate Playwright to `http://localhost:8765/{page-path}`
2. Read console messages: `mcp__playwright__browser_console_messages → level: error`
3. Extract all 404 URLs from console errors
4. Download each missing asset from the original domain
5. Reload and repeat until **console errors = 0**
6. Only "Slow network" font warnings are acceptable (not real errors)

### Step 7: Generate manifest

Write `site-manifest.json` to the clone directory:

```json
{
  "source": "{original_url}",
  "cloned_at": "{timestamp}",
  "total_files": N,
  "total_size_bytes": N,
  "pages": ["/", "/about", "..."],
  "assets": {
    "images": N,
    "fonts": N,
    "scripts": N,
    "styles": N
  },
  "missing": []
}
```

### Step 8: Screenshot comparison

Take a full-page screenshot of the local clone and report back.

## Stop conditions

- **DONE**: 0 console errors, visual match confirmed
- **DONE_WITH_CONCERNS**: ≤ 2 minor missing assets that don't affect layout (e.g. favicon)
- **BLOCKED**: Original site requires login / CAPTCHA / bot detection that we can't bypass

## Output

After completion, tell the user:
- Local URL: `http://localhost:8765/{page-path}`
- Clone directory path
- File count and total size
- Any assets that couldn't be downloaded and why

## Cleanup

Do NOT delete the clone directory or server — leave them running for the user to inspect.
