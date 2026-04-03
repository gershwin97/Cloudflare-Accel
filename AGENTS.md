# AGENTS.md - Cloudflare-Accel Developer Guide

## Project Overview

This is a Cloudflare Workers/Pages project providing GitHub and Docker acceleration services via reverse proxy. The core code is in `_worker.js` - a single JavaScript file that handles both the web UI and proxy functionality.

## Build, Lint, and Test Commands

### Testing the Worker (curl-based)

```bash
# Test homepage
curl https://d.gershwin.link/

# Test GitHub file proxy (returns 200 with direct content)
curl -I https://d.gershwin.link/https://github.com/cloudflare/cloudflared/releases/download/2025.7.0/cloudflared-linux-amd64

# Test Docker image proxy
curl -I https://d.gershwin.link/nginx

# Test invalid domain (should return 400)
curl https://d.gershwin.link/invalid.com/path
```

### Deployment Commands

```bash
# Deploy via Cloudflare Dashboard - copy _worker.js content to Worker editor
# Or use Wrangler CLI
npx wrangler deploy _worker.js
```

### Linting

```bash
npx eslint _worker.js
```

### No Build Required

This project is pure JavaScript - no build step needed. Deploy `_worker.js` directly to Cloudflare Workers/Pages.

## Code Style Guidelines

### General Style

- Use ES6+ JavaScript syntax
- Cloudflare Workers use Service Worker syntax with `export default`
- 2-space indentation
- No semicolons at end of statements
- Use template literals for string interpolation

### Imports and Dependencies

Cloudflare Workers use native `fetch`, no imports needed. External dependencies via CDN in HTML (e.g., Tailwind CSS).

### Naming Conventions

- **Constants**: UPPER_SNAKE_CASE (e.g., `ALLOWED_HOSTS`, `RESTRICT_PATHS`)
- **Variables/functions**: camelCase (e.g., `handleRequest`, `parseDockerImage`)
- **HTML strings**: UPPER_SNAKE_CASE (e.g., `HOMEPAGE_HTML`, `LIGHTNING_SVG`)

### Types

- Use `const` for immutable values
- Use `let` for mutable values
- Avoid `var`
- All config values are arrays of strings or booleans

### Error Handling

Return error responses with appropriate HTTP status codes:

```javascript
return new Response(`Error: ${errorMessage}`, {
  status: 400,  // 400 for bad request, 403 for forbidden, 404 for not found
  headers: { 'Content-Type': 'text/plain; charset=utf-8' }
});
```

### Response Format

```javascript
// HTML responses
return new Response(htmlContent, {
  headers: { 'Content-Type': 'text/html; charset=utf-8' }
});

// Proxy responses - pass through with appropriate headers
return new Response(response.body, {
  status: response.status,
  headers: response.headers
});
```

### Function Structure

```javascript
// Main handler - must be default export
export default {
  async fetch(request, env, ctx) {
    return await handleRequest(request);
  }
};

// Helper functions defined outside
async function handleRequest(request) {
  const url = new URL(request.url);
}
```

### Configuration Section

Keep user-configurable options in a clearly marked section at the top of the file:

```javascript
// 用户配置区域开始 =================================
const ALLOWED_HOSTS = [...];
const RESTRICT_PATHS = false;
const ALLOWED_PATHS = [...];
// 用户配置区域结束 =================================
```

### String Literals

- Use double quotes for HTML attributes
- Use backticks for template literals with variables
- Use single quotes for simple strings

### Comments

- Use Chinese comments for user-facing configuration guidance
- Keep comments concise and informative

### Best Practices

1. **Security**: Validate all input domains against whitelist
2. **Performance**: Reuse Response objects when possible
3. **Compatibility**: Use module syntax (`export default`) for Cloudflare Pages
4. **Redirects**: Handle 302/307 redirects recursively for Docker
5. **Headers**: Pass through relevant headers (content-type, content-length, etc.)

### Common Patterns

```javascript
// Parse URL and extract path
const url = new URL(request.url);
const pathname = url.pathname;

// Validate domain
if (!ALLOWED_HOSTS.includes(hostname)) {
  return new Response('Error: Invalid target domain.', { status: 400 });
}

// Handle Docker image shorthand
function parseDockerImage(input) {
  if (!input.includes('/')) {
    return `library/${input}`;  // Official image
  }
  return input;
}
```

## File Structure

```
├── _worker.js      # Main worker code (UI + proxy)
├── README.md       # Documentation
├── AGENTS.md       # Developer guide (this file)
└── LICENSE         # MIT License
```

## Key Configuration Variables

| Variable | Type | Description |
|----------|------|-------------|
| `ALLOWED_HOSTS` | string[] | Whitelisted domains for proxy |
| `RESTRICT_PATHS` | boolean | Enable path restrictions |
| `ALLOWED_PATHS` | string[] | Allowed path keywords when RESTRICT_PATHS=true |

## Testing Best Practices

1. Test with both HTTP and HTTPS GitHub links
2. Test Docker images with and without registry prefix
3. Test path restriction mode (RESTRICT_PATHS=true)
4. Test invalid domains return 400 errors
5. Test mobile UI responsiveness and clipboard copy

## Common Tasks

### Adding a new allowed domain

Edit `ALLOWED_HOSTS` array in the configuration section.

### Adding path restriction

1. Set `RESTRICT_PATHS = true`
2. Add keywords to `ALLOWED_PATHS` array

### Modifying the UI

Edit `HOMEPAGE_HTML` constant - uses Tailwind CSS for styling.

## Basic Logic Summary

The worker handles requests in this order:

### 1. Homepage Route
- Path `/` or empty → returns `HOMEPAGE_HTML` (the web UI)

### 2. URL Parsing
Extracts target domain and path from request path in multiple formats:
- `/https://github.com/user/repo/file` - explicit URL with protocol
- `/nginx` - Docker shorthand (converts to `library/nginx`)
- `/ghcr.io/user/image` - explicit Docker registry
- `/v2/library/nginx/manifests/latest` - Docker V2 API

### 3. Domain Validation
- Checks `targetDomain` against `ALLOWED_HOSTS` whitelist → 400 if invalid

### 4. Path Validation
- If `RESTRICT_PATHS = true`, checks path against `ALLOWED_PATHS` → 403 if not allowed

### 5. Proxy Flow
- Constructs target URL (`https://targetDomain/path`)
- For Docker requests with 401: acquires Bearer token from realm
- Handles 307/302 redirects recursively (for Docker image layers on AWS S3)
- Proxies request with cleaned headers
- Adds CORS headers to response
- For Docker: removes Location header to prevent direct client redirects

### 6. AWS S3 Handling
- Adds `x-amz-content-sha256` and `x-amz-date` headers for S3 targets
- Ensures requests work with AWS signature authentication