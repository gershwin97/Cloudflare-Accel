# AGENTS.md - Cloudflare-Accel Developer Guide

## Project Overview

Cloudflare Workers/Pages project providing GitHub and Docker acceleration via reverse proxy. Core code is `_worker.js` (single file with UI + proxy).

## Build, Lint, and Test Commands

### Testing with curl
```bash
# Test homepage
curl https://d.gershwin.link/

# Test GitHub file proxy
curl -I https://d.gershwin.link/https://github.com/cloudflare/cloudflared/releases/download/2025.7.0/cloudflared-linux-amd64

# Test Docker image proxy
curl -I https://d.gershwin.link/nginx

# Test invalid domain (should return 400)
curl https://d.gershwin.link/invalid.com/path
```

### Deployment
```bash
# Via Cloudflare Dashboard - copy _worker.js to Worker editor
# Or via Wrangler CLI
npx wrangler deploy _worker.js
```

### Linting
```bash
npx eslint _worker.js
```

### No Build Required

Pure JavaScript - deploy `_worker.js` directly.

## Code Style Guidelines

### General
- ES6+ JavaScript syntax
- 2-space indentation, no trailing semicolons
- Use template literals for string interpolation
- Use `const` for immutable values, `let` for mutable, avoid `var`

### Imports and Dependencies
- Cloudflare Workers use native `fetch`, no imports needed
- External dependencies via CDN in HTML (e.g., Tailwind CSS)

### Naming Conventions
- **Constants**: UPPER_SNAKE_CASE (e.g., `ALLOWED_HOSTS`, `RESTRICT_PATHS`)
- **Variables/functions**: camelCase (e.g., `handleRequest`, `parseDockerImage`)
- **HTML strings**: UPPER_SNAKE_CASE (e.g., `HOMEPAGE_HTML`, `LIGHTNING_SVG`)

### String Literals
- Double quotes for HTML attributes
- Backticks for template literals with variables
- Single quotes for simple strings

### Comments
- Use Chinese comments for user-facing configuration guidance
- Keep comments concise and informative

### Error Handling

Return error responses with appropriate HTTP status codes:
```javascript
return new Response(`Error: ${errorMessage}`, {
  status: 400,  // 400 bad request, 403 forbidden, 404 not found
  headers: { 'Content-Type': 'text/plain; charset=utf-8' }
});
```

### Response Format

```javascript
// HTML responses
return new Response(htmlContent, {
  headers: { 'Content-Type': 'text/html; charset=utf-8' }
});

// Proxy responses - pass through with cleaned headers
return new Response(response.body, {
  status: response.status,
  headers: response.headers
});
```

### Function Structure

```javascript
// Main handler - must use default export
export default {
  async fetch(request, env, ctx) {
    return await handleRequest(request)
  }
}

// Helper functions defined outside
async function handleRequest(request, redirectCount = 0) {
  const url = new URL(request.url)
}
```

### Configuration Section

Keep user-configurable options at the top of the file:
```javascript
// 用户配置区域开始 =================================
const ALLOWED_HOSTS = [...]
const RESTRICT_PATHS = true
const ALLOWED_PATHS = [...]
// 用户配置区域结束 =================================
```

## File Structure
```
├── _worker.js      # Main worker code (UI + proxy)
├── README.md       # Documentation
├── AGENTS.md       # Developer guide
└── LICENSE         # MIT License
```

## Key Configuration Variables

| Variable | Type | Description |
|----------|------|-------------|
| `ALLOWED_HOSTS` | string[] | Whitelisted domains for proxy |
| `RESTRICT_PATHS` | boolean | Enable path restrictions |
| `ALLOWED_PATHS` | string[] | Allowed path keywords |

## Common Patterns

```javascript
// Parse URL and extract path
const url = new URL(request.url)
const pathname = url.pathname

// Domain validation
if (!ALLOWED_HOSTS.includes(targetDomain)) {
  return new Response('Error: Invalid target domain.', { status: 400 })
}

// Handle Docker image shorthand
if (!input.includes('/')) {
  return `library/${input}`
}
```

## Request Flow

1. **Homepage**: Path `/` or empty → returns `HOMEPAGE_HTML`
2. **URL Parsing**: Extract domain/path from formats like `/https://github.com/...`, `/nginx`, `/ghcr.io/user/image`
3. **Domain Validation**: Check against `ALLOWED_HOSTS` whitelist → 400 if invalid
4. **Path Validation**: If `RESTRICT_PATHS=true`, check against `ALLOWED_PATHS` → 403 if not allowed
5. **Proxy**: Handle 401 auth, 307/302 redirects, add CORS headers, strip Location for Docker

## Best Practices

1. Validate all input domains against whitelist
2. Handle 302/307 redirects recursively for Docker image layers
3. Add `x-amz-content-sha256` and `x-amz-date` headers for AWS S3 targets
4. Remove `Location` header for Docker to prevent client-side redirects
5. Add CORS headers: `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, HEAD, POST, OPTIONS`
