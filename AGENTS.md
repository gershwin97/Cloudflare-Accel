# AGENTS.md - Cloudflare-Accel Developer Guide

## Project Overview

Cloudflare Workers project providing GitHub and Docker acceleration via reverse proxy. Single file `_worker.js` contains all code (UI + proxy logic).

## Build, Lint, and Test Commands

### No Build Required
Pure JavaScript - deploy `_worker.js` directly via Cloudflare Dashboard or Wrangler CLI.

### Deployment
```bash
# Via Cloudflare Dashboard - copy _worker.js content to Worker editor
# Or via Wrangler CLI
npx wrangler deploy _worker.js
```

### Linting
```bash
npx eslint _worker.js
```

### Testing with curl
```bash
# Test homepage
curl https://d.gershwin.link/

# Test GitHub file proxy
curl -I https://d.gershwin.link/https://github.com/cloudflare/cloudflared/releases/download/2025.7.0/cloudflared-linux-amd64

# Test GitHub API proxy
curl -I https://d.gershwin.link/api.github.com/repos/cloudflare/cloudflared/releases/latest

# Test Docker image proxy (shorthand)
curl -I https://d.gershwin.link/nginx

# Test Docker image proxy (full path)
curl -I https://d.gershwin.link/ghcr.io/homebrew/brew

# Test Docker V2 API manifests
curl -I https://d.gershwin.link/v2/library/nginx/manifests/latest

# Test Docker V2 API blobs
curl -I https://d.gershwin.link/v2/library/nginx/blobs/sha256:abc123

# Test invalid domain (should return 400)
curl https://d.gershwin.link/invalid.com/path
```

### No Test Framework
This project has no automated tests. All validation is done via manual curl testing.

## Code Style Guidelines

### General
- ES6+ JavaScript syntax
- 2-space indentation, no trailing semicolons
- Template literals for string interpolation
- `const` for immutable values, `let` for mutable, avoid `var`

### Imports and Dependencies
- Cloudflare Workers use native `fetch` and `crypto` APIs - no imports needed
- External dependencies via CDN in HTML (e.g., Tailwind CSS from cdn.tailwindcss.com)

### Naming Conventions
| Type | Convention | Example |
|------|------------|---------|
| Constants | UPPER_SNAKE_CASE | `ALLOWED_HOSTS`, `RESTRICT_PATHS` |
| Variables/functions | camelCase | `handleRequest`, `parseDockerImage` |
| HTML strings | UPPER_SNAKE_CASE | `HOMEPAGE_HTML`, `LIGHTNING_SVG` |
| Private helpers | camelCase with `is`/`handle` prefix | `isAmazonS3`, `handleToken` |

### String Literals
- Double quotes for HTML attributes: `class="flex-grow"`
- Backticks for template literals with variables: `` `https://${domain}/${path}` ``
- Single quotes for simple strings: `'GET'`, `'Accept'`

### Comments
- Use Chinese comments for user-facing configuration guidance
- Use English comments for code logic explanation
- Keep comments concise, avoid obvious statements

### Error Handling
Return errors with appropriate HTTP status codes:
```javascript
return new Response(`Error: ${errorMessage}`, {
  status: 400,  // 400=bad request, 403=forbidden, 404=not found, 500=server error
  headers: { 'Content-Type': 'text/plain; charset=utf-8' }
});
```

### Response Formats
```javascript
// HTML responses
return new Response(htmlContent, {
  headers: { 'Content-Type': 'text/html; charset=utf-8' }
})

// Proxy responses - pass through with CORS headers
const newResponse = new Response(response.body, response)
newResponse.headers.set('Access-Control-Allow-Origin', '*')
newResponse.headers.set('Access-Control-Allow-Methods', 'GET, HEAD, POST, OPTIONS')
return newResponse
```

### Function Structure
```javascript
export default {
  async fetch(request, env, ctx) {
    return handleRequest(request)
  }
}

async function handleRequest(request, redirectCount = 0) {
  const url = new URL(request.url)
  const MAX_REDIRECTS = 5
  // ...
}
```

## Configuration Section

User-configurable options at top of file:
```javascript
// 用户配置区域开始 =================================
const ALLOWED_HOSTS = [
  'github.com',
  'api.github.com',
  'raw.githubusercontent.com',
]

const RESTRICT_PATHS = true
const ALLOWED_PATHS = ['library', 'senshinya', 'gershwin97']
// 用户配置区域结束 =================================
```

## File Structure
```
├── _worker.js      # Main worker code (UI + proxy, 737 lines)
├── README.md       # Documentation
├── AGENTS.md       # Developer guide (this file)
└── LICENSE         # MIT License
```

## Request Flow

1. **Homepage**: Path `/` or empty → returns `HOMEPAGE_HTML`
2. **URL Parsing**: Extract domain/path from `/https://github.com/...`, `/nginx`, `/ghcr.io/user/image`, `/v2/...`
3. **Domain Validation**: Check against `ALLOWED_HOSTS` whitelist → 400 if invalid
4. **Path Validation**: If `RESTRICT_PATHS=true`, check against `ALLOWED_PATHS` → 403 if not allowed
5. **Proxy**: Handle 401 auth, recursive 307/302 redirects, add CORS headers, strip Location for Docker

## Key Helper Functions

| Function | Purpose |
|----------|---------|
| `isAmazonS3(url)` | Check if URL is AWS S3 |
| `calculateSHA256(message)` | Compute SHA256 hash for AWS S3 auth |
| `handleToken(realm, service, scope)` | Fetch Docker registry auth token |
| `handleRequest(request, redirectCount)` | Main request handler with redirect logic |

## Best Practices

1. Validate all input domains against whitelist before proxying
2. Handle 302/307 redirects recursively (up to MAX_REDIRECTS=5) for Docker layers
3. Add `x-amz-content-sha256` and `x-amz-date` headers for AWS S3 requests
4. Remove `Location` header for Docker to prevent client-side redirects
5. Add CORS headers: `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, HEAD, POST, OPTIONS`
