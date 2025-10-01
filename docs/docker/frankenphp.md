# FrankenPHP Multi-Tenant Caddyfile Configuration

This directory contains optimized Caddyfile configurations for the ACME Corp CSR Platform's multi-tenant architecture using FrankenPHP and Laravel Octane.

## Files Overview

- **`Caddyfile.dev`** - Development configuration with hot reloading and debug features
- **`Caddyfile.prod`** - Production configuration with HTTPS, security headers, and performance optimizations
- **`README.md`** - This documentation file

## Architecture Overview

### Multi-Tenant Structure

The platform supports two types of domains:

1. **Central Domains** (Admin/Management)
   - `localhost`, `127.0.0.1` (development)
   - `acme-corp-optimy.test` (development)
   - `yourdomain.com` (production)

2. **Tenant Domains** (Customer-facing)
   - `*.localhost` (development)
   - `*.app.yourdomain.com` (production)

## Development Configuration (`Caddyfile.dev`)

### Features

- **Hot Reloading**: Supports Vite development server integration
- **Debug Logging**: Comprehensive logging for development debugging
- **CORS**: Permissive CORS settings for frontend development
- **Tenant Context**: Automatic tenant identification from subdomains
- **Admin Protection**: Blocks admin routes on tenant domains

### Environment Variables

No environment variables required for development - defaults to localhost/127.0.0.1.

### Usage

```bash
# Docker environment
docker-compose up -d

# Access points:
# Central app: http://localhost, http://127.0.0.1
# Tenant example: http://tenant1.localhost
```

## Production Configuration (`Caddyfile.prod`)

### Features

- **Automatic HTTPS**: Let's Encrypt SSL certificates for all domains
- **HTTP/2 & HTTP/3**: Modern protocol support
- **Security Headers**: HSTS, CSP, CSRF protection
- **Rate Limiting**: Multi-tier rate limiting (admin, API, tenant)
- **Performance**: Advanced compression (Zstandard, Brotli)
- **Monitoring**: Health checks, metrics endpoints
- **Tenant Isolation**: Separate logging and rate limits per tenant

### Required Environment Variables

```bash
# Production domain configuration
CENTRAL_DOMAIN=yourdomain.com              # Central app domain
TENANT_DOMAIN=app.yourdomain.com           # Tenant base domain
ACME_EMAIL=admin@yourdomain.com           # Let's Encrypt email

# Mercure configuration (optional)
MERCURE_PUBLISHER_JWT_KEY=# Set in .env
MERCURE_SUBSCRIBER_JWT_KEY=# Set in .env
```

### Domain Examples

With the above configuration:
- Central: `https://yourdomain.com` (admin panel, management)
- Tenant: `https://client1.app.yourdomain.com` (client-facing app)
- Tenant: `https://nonprofit1.app.yourdomain.com` (another client)

## Security Features

### Central App Security

- **HSTS Preload**: Force HTTPS with preload directive
- **CSP**: Strict Content Security Policy
- **Admin Rate Limiting**: 30 requests/minute for admin routes
- **API Rate Limiting**: 1000 requests/minute, 10/minute for auth endpoints
- **File Protection**: Blocks access to sensitive files (.env, .git, etc.)

### Tenant App Security

- **Tenant Isolation**: Separate rate limits and logging per tenant
- **Admin Blocking**: Prevents admin access on tenant domains
- **CORS**: Restricted CORS for tenant-specific origins
- **Asset Protection**: Tenant-specific storage isolation

## Performance Optimizations

### Compression

- **Zstandard**: Level 22 (central), Level 15 (tenant)
- **Brotli**: Level 11 (central), Level 8 (tenant)
- **Gzip**: Level 9 (central), Level 6 (tenant)

### Caching

- **Static Assets**: 1-year cache with immutable directive
- **API Responses**: No-cache for dynamic content
- **Tenant Assets**: 24-hour cache for tenant-specific assets

### FrankenPHP Workers

- **Development**: 4 workers with debugging enabled
- **Production**: 32 workers optimized for multi-tenant load

## Rate Limiting

### Central App

| Zone | Limit | Window | Purpose |
|------|--------|---------|----------|
| admin | 30 req | 1 min | Admin panel protection |
| api | 1000 req | 1 min | General API access |
| auth | 10 req | 1 min | Authentication endpoints |

### Tenant Apps

| Zone | Limit | Window | Purpose |
|------|--------|---------|----------|
| tenant_api | 500 req | 1 min | Per-tenant API limits |
| suspicious | 3 req | 1 hour | Block malicious requests |

## Monitoring & Logging

### Health Checks

- **Central**: `GET /health` - Basic health check
- **Central Detailed**: `GET /health/detailed` - Internal networks only
- **Tenant**: `GET /health` - Returns tenant ID

### Metrics

- **Central**: `GET /metrics` - Prometheus format (internal only)
- **Logs**: JSON format with rotation (90 days central, 30 days tenant)

### Log Files

```
/var/log/caddy/
├── central-access.log       # Central app logs
├── tenant-{name}-access.log # Per-tenant logs
└── error.log               # Global error logs
```

## Tenant Context Environment Variables

The Caddyfile automatically sets these environment variables for Laravel:

### Central Domains
```bash
TENANT_CENTRAL=true
SERVER_NAME=yourdomain.com
```

### Tenant Domains
```bash
TENANT_CENTRAL=false
TENANT_SUBDOMAIN=client1                    # Extracted from subdomain
TENANT_DOMAIN=client1.app.yourdomain.com    # Full tenant domain
SERVER_NAME=client1.app.yourdomain.com      # Full host
```

## Laravel Integration

Your Laravel application can access tenant context:

```php
// In your Laravel middleware or service provider
$isCentral = env('TENANT_CENTRAL') === 'true';
$tenantId = env('TENANT_SUBDOMAIN');
$serverName = env('SERVER_NAME');

if (!$isCentral && $tenantId) {
    // Initialize tenant context
    tenancy()->initialize($tenantId);
}
```

## DNS Configuration

### Development

Add to `/etc/hosts`:
```
127.0.0.1 localhost acme-corp-optimy.test
127.0.0.1 tenant1.localhost tenant2.localhost
```

### Production

**Central Domain (A/AAAA records):**
```
yourdomain.com.          300   IN  A      YOUR_SERVER_IP
www.yourdomain.com.      300   IN  CNAME  yourdomain.com.
```

**Wildcard Tenant Domain:**
```
*.app.yourdomain.com.    300   IN  A      YOUR_SERVER_IP
app.yourdomain.com.      300   IN  A      YOUR_SERVER_IP
```

## Deployment

### Docker Compose Production

```yaml
services:
  app:
    build:
      target: frankenphp_prod
    environment:
      CENTRAL_DOMAIN: yourdomain.com
      TENANT_DOMAIN: app.yourdomain.com
      ACME_EMAIL: admin@yourdomain.com
      MERCURE_PUBLISHER_JWT_KEY: ${MERCURE_PUBLISHER_JWT_KEY}
      MERCURE_SUBSCRIBER_JWT_KEY: ${MERCURE_SUBSCRIBER_JWT_KEY}
```

### SSL Certificate Management

Caddy automatically manages SSL certificates:
- Let's Encrypt certificates for all domains
- Automatic renewal (60 days before expiry)
- HTTP to HTTPS redirects
- Wildcard certificate support for tenant domains

## Troubleshooting

### Common Issues

1. **SSL Certificate Failures**
   - Ensure DNS is properly configured
   - Check ACME_EMAIL is valid
   - Verify domains are publicly accessible

2. **Tenant Not Loading**
   - Check subdomain DNS configuration
   - Verify wildcard certificate is issued
   - Check tenant exists in Laravel application

3. **Rate Limiting Triggered**
   - Check rate limit zones in logs
   - Adjust limits if necessary for your use case
   - Implement proper API authentication

### Debug Commands

```bash
# Check Caddy configuration
docker exec acme-app caddy validate --config /etc/caddy/Caddyfile

# View Caddy logs
docker logs acme-app

# Check certificate status
docker exec acme-app caddy list-certificates
```

## Performance Tuning

### For High Traffic

Adjust in production Caddyfile:

```caddyfile
frankenphp {
    worker {
        num 64  # Increase workers
    }
}

# Increase rate limits
rate_limit {
    zone api {
        events 5000  # Higher API limits
    }
}
```

### For Many Tenants

Consider log rotation adjustments:

```caddyfile
log {
    output file /var/log/caddy/tenant-{labels.3}-access.log {
        roll_size 50mb   # Smaller files
        roll_keep 5      # Fewer kept files
        roll_keep_for 168h  # 7 days retention
    }
}
```

## Security Considerations

1. **Environment Variables**: Store sensitive configs in Docker secrets
2. **Rate Limiting**: Monitor and adjust based on legitimate traffic patterns  
3. **SSL**: Enable HSTS preload only after testing
4. **CSP**: Customize Content Security Policy for your frontend needs
5. **Logging**: Ensure log files don't contain sensitive information

This configuration provides enterprise-grade multi-tenant hosting with security, performance, and monitoring built-in.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved