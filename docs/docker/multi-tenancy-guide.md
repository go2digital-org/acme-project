# ACME Corp CSR Platform - Multi-Tenancy Guide

## Overview

The ACME Corp CSR Platform implements a robust multi-tenancy architecture using database-per-tenant isolation combined with subdomain routing. This guide covers the setup, management, and operation of multi-tenant environments using Docker.

## Table of Contents

1. [Architecture](#architecture)
2. [Quick Start](#quick-start)
3. [Tenant Management](#tenant-management)
4. [Domain Configuration](#domain-configuration)
5. [Database Isolation](#database-isolation)
6. [Queue & Cache Separation](#queue--cache-separation)
7. [Search Index Management](#search-index-management)
8. [Storage Isolation](#storage-isolation)
9. [Security Considerations](#security-considerations)
10. [Troubleshooting](#troubleshooting)

## Architecture

### Multi-Tenancy Model

The platform uses a **hybrid multi-tenancy model**:

- **Database-per-tenant**: Each tenant has its own isolated database
- **Shared application**: Single application instance serves all tenants
- **Subdomain routing**: Tenants access via unique subdomains
- **Resource isolation**: Separate storage, cache keys, and search indexes

### Components

```
┌─────────────────────────────────────────┐
│         FrankenPHP + Octane             │
│     (Shared Application Instance)        │
├─────────────────────────────────────────┤
│          Subdomain Router               │
│  ┌─────────┬──────────┬──────────┐     │
│  │tenant1. │ tenant2. │ tenant3. │     │
└──┴─────────┴──────────┴──────────┴─────┘
         │          │           │
    ┌────▼────┬─────▼─────┬────▼────┐
    │tenant_1 │ tenant_2  │tenant_3 │  MySQL Databases
    └─────────┴───────────┴─────────┘

    ┌─────────────────────────────────┐
    │      Redis (Namespaced)         │  Cache & Queues
    │  tenant:1:* | tenant:2:*        │
    └─────────────────────────────────┘

    ┌─────────────────────────────────┐
    │    Meilisearch (Indexed)        │  Search
    │  tenant_1_* | tenant_2_*        │
    └─────────────────────────────────┘
```

## Quick Start

### 1. Start the Platform

```bash
# Start Docker environment
./docker/scripts/docker-manager.sh start

# Verify all services are healthy
./docker/scripts/docker-manager.sh health
```

### 2. Create Your First Tenant

```bash
# Create a tenant with ID 'acme-corp'
./docker/scripts/tenant-manager.sh create acme-corp \
    "ACME Corporation" \
    "localhost" \
    "admin@localhost"
```

### 3. Access the Tenant

Add to your `/etc/hosts` file:
```
127.0.0.1 localhost
```

Visit: http://localhost:8000

## Tenant Management

### Creating Tenants

#### Basic Creation
```bash
./docker/scripts/tenant-manager.sh create <tenant_id>
```

#### Full Configuration
```bash
./docker/scripts/tenant-manager.sh create <tenant_id> \
    "<Tenant Name>" \
    "<domain>" \
    "<admin_email>"
```

#### Example
```bash
# Create a production tenant
./docker/scripts/tenant-manager.sh create unicef \
    "UNICEF Foundation" \
    "tenant.yourdomain.com" \
    "admin@tenant.org"
```

### Listing Tenants

```bash
# List all tenants with database info
./docker/scripts/tenant-manager.sh list

# Example output:
# ID          Database            Size
# ----------------------------------------
# acme-corp   tenant_acme-corp    45.2 MB
# unicef      tenant_unicef       12.8 MB
# redcross    tenant_redcross     28.5 MB
# ----------------------------------------
# Total tenants: 3
```

### Viewing Tenant Information

```bash
./docker/scripts/tenant-manager.sh info <tenant_id>

# Shows:
# - Database details
# - Storage usage
# - Search indexes
# - Table count
```

### Deleting Tenants

```bash
# Interactive deletion (with confirmation)
./docker/scripts/tenant-manager.sh delete <tenant_id>

# Force deletion (no confirmation)
./docker/scripts/tenant-manager.sh delete <tenant_id> --force
```

### Backing Up Tenants

```bash
# Backup to default location
./docker/scripts/tenant-manager.sh backup <tenant_id>

# Backup to specific directory
./docker/scripts/tenant-manager.sh backup <tenant_id> /path/to/backups
```

### Restoring Tenants

```bash
# Restore with same tenant ID
./docker/scripts/tenant-manager.sh restore /path/to/backup

# Restore as new tenant
./docker/scripts/tenant-manager.sh restore /path/to/backup new-tenant-id
```

## Domain Configuration

### Development Domains

For local development, tenants use subdomains of `localhost`:

```bash
# Pattern: <tenant-id>.localhost
localhost
unicef.localhost
redcross.localhost
```

Add entries to `/etc/hosts`:
```bash
127.0.0.1 localhost
127.0.0.1 unicef.localhost
127.0.0.1 redcross.localhost
```

### Production Domains

For production, configure DNS with wildcard subdomain:

#### DNS Configuration
```dns
# Main domain
yourdomain.com.        A    YOUR_SERVER_IP

# Wildcard for all tenants
*.yourdomain.com.      A    YOUR_SERVER_IP
```

#### SSL/TLS Setup
The platform automatically handles SSL certificates via Let's Encrypt:

```yaml
# In docker-compose.prod.yml
environment:
  SERVER_NAME: "yourdomain.com, *.yourdomain.com:443"
  LETS_ENCRYPT_EMAIL: admin@yourdomain.com
```

### Central vs Tenant Domains

**Central Domains** (Admin access):
- `localhost`
- `yourdomain.com`
- `admin.yourdomain.com`

**Tenant Domains** (Tenant access):
- `tenant-id.localhost`
- `tenant-id.yourdomain.com`

## Database Isolation

### Database Naming Convention

Each tenant database follows the pattern:
```
tenant_<tenant_id>
```

Examples:
- `tenant_acme_corp`
- `tenant_unicef`
- `tenant_redcross`

### Running Migrations

```bash
# Migrate specific tenant
./docker/scripts/tenant-manager.sh migrate <tenant_id>

# Rollback migrations
./docker/scripts/tenant-manager.sh migrate <tenant_id> --rollback

# Migrate all tenants
./docker/scripts/tenant-manager.sh run-all "php artisan migrate --force"
```

### Database Access

```bash
# Access tenant database
docker exec -it acme-mysql mysql -uroot -proot tenant_<tenant_id>

# Run SQL query for tenant
docker exec acme-mysql mysql -uroot -proot \
    -e "SELECT * FROM tenant_acme_corp.users LIMIT 5;"
```

## Queue & Cache Separation

### Queue Isolation

Each tenant's jobs are tagged with tenant ID:

```php
// Jobs are automatically tagged
dispatch(new ProcessDonation($donation))
    ->onQueue("tenant:{$tenantId}:payments");
```

### Queue Workers

Supervisor manages tenant-aware queue workers:

```ini
[program:queue-tenant-high]
command=php artisan queue:work redis
    --queue=tenant:%(ENV_TENANT_ID)s:payments,tenant:%(ENV_TENANT_ID)s:notifications
    --tries=5
    --memory=256
```

### Cache Namespacing

Cache keys are automatically prefixed:

```php
// Cached as: tenant:acme-corp:campaigns:all
Cache::remember('campaigns:all', 3600, function() {
    return Campaign::all();
});
```

### Clearing Tenant Cache

```bash
# Clear specific tenant cache
docker exec acme-app php artisan cache:clear --tenant=<tenant_id>

# Clear all tenant caches
./docker/scripts/tenant-manager.sh run-all "php artisan cache:clear"
```

## Search Index Management

### Index Naming Convention

Meilisearch indexes follow the pattern:
```
tenant_<tenant_id>_<entity>
```

Examples:
- `tenant_acme_corp_campaigns`
- `tenant_acme_corp_donations`
- `tenant_unicef_campaigns`

### Creating Indexes

Indexes are automatically created during tenant creation:

```bash
# Manual index creation for tenant
docker exec acme-app bash -c \
    "TENANT_ID=acme-corp bash /app/scripts/meilisearch-setup.sh"
```

### Rebuilding Indexes

```bash
# Rebuild specific tenant's search indexes
docker exec acme-app php artisan scout:import \
    "App\Models\Campaign" --tenant=<tenant_id>
```

## Storage Isolation

### Directory Structure

Each tenant has isolated storage:
```
storage/app/tenants/
├── acme-corp/
│   ├── uploads/
│   ├── exports/
│   ├── cache/
│   └── logs/
├── unicef/
│   ├── uploads/
│   ├── exports/
│   ├── cache/
│   └── logs/
```

### Storage Management

```bash
# Check tenant storage usage
du -sh storage/app/tenants/<tenant_id>

# Clean tenant temporary files
find storage/app/tenants/<tenant_id>/cache -mtime +7 -delete
```

## Security Considerations

### 1. Database Isolation

- Each tenant has a completely separate database
- No cross-database queries allowed
- Separate database users per tenant (optional)

### 2. Application-Level Security

```php
// Middleware ensures tenant isolation
middleware(['tenant.aware', 'tenant.verify'])
```

### 3. API Rate Limiting

Per-tenant rate limits:
```php
RateLimiter::for('api', function (Request $request) {
    $tenant = tenant();
    return Limit::perMinute(60)->by($tenant?->id ?: $request->ip());
});
```

### 4. Admin Access Control

Admin routes blocked on tenant domains:
```nginx
# Caddyfile configuration
@admin {
    path /admin/* /horizon/* /telescope/*
}
handle @admin {
    @notTenant {
        host localhost 127.0.0.1 acme-corp-optimy.test
    }
    # Block on tenant subdomains
    respond "Admin access not available on tenant domains" 403
}
```

## Troubleshooting

### Common Issues

#### 1. Tenant Domain Not Accessible

**Problem**: Cannot access tenant subdomain

**Solutions**:
```bash
# Check /etc/hosts entry
grep "tenant-id.localhost" /etc/hosts

# Verify DNS (production)
nslookup tenant-id.yourdomain.com

# Check Caddy configuration
docker exec acme-app cat /etc/caddy/Caddyfile | grep SERVER_NAME
```

#### 2. Database Connection Issues

**Problem**: Tenant database connection failed

**Solutions**:
```bash
# Verify database exists
docker exec acme-mysql mysql -uroot -proot \
    -e "SHOW DATABASES LIKE 'tenant_%';"

# Check database permissions
docker exec acme-mysql mysql -uroot -proot \
    -e "SHOW GRANTS FOR 'acme'@'%';"

# Test connection
docker exec acme-app php artisan tinker \
    --execute="DB::connection('tenant')->getPdo();"
```

#### 3. Queue Workers Not Processing

**Problem**: Tenant jobs stuck in queue

**Solutions**:
```bash
# Check worker status
docker exec acme-app supervisorctl status | grep queue

# Restart workers
docker exec acme-app supervisorctl restart queue-high-priority:*

# Check Redis queues
docker exec acme-redis redis-cli KEYS "queues:tenant:*"
```

#### 4. Search Not Working

**Problem**: Search returns no results for tenant

**Solutions**:
```bash
# Check indexes exist
curl -H "Authorization: Bearer masterKey" \
    http://localhost:7700/indexes | jq

# Rebuild tenant indexes
docker exec acme-app php artisan scout:flush "App\Models\Campaign" \
    --tenant=<tenant_id>
docker exec acme-app php artisan scout:import "App\Models\Campaign" \
    --tenant=<tenant_id>
```

### Debug Commands

```bash
# View tenant environment variables
docker exec acme-app env | grep TENANT

# Check active tenant in application
docker exec acme-app php artisan tinker \
    --execute="echo tenant()->id ?? 'No tenant';"

# Monitor tenant-specific logs
docker exec acme-app tail -f storage/app/tenants/<tenant_id>/logs/laravel.log

# Database query debugging
docker exec acme-mysql mysql -uroot -proot \
    -e "SHOW PROCESSLIST;" | grep tenant_
```

## Advanced Operations

### Bulk Operations

```bash
# Run artisan command for all tenants
./docker/scripts/tenant-manager.sh run-all "php artisan cache:clear"

# Backup all tenants
for tenant in $(./docker/scripts/tenant-manager.sh list | grep tenant_ | awk '{print $1}'); do
    ./docker/scripts/tenant-manager.sh backup $tenant
done
```

### Tenant Cloning

```bash
# Backup source tenant
./docker/scripts/tenant-manager.sh backup source-tenant

# Restore as new tenant
./docker/scripts/tenant-manager.sh restore \
    backups/tenants/source-tenant/[timestamp] \
    new-tenant
```

### Performance Monitoring

```bash
# Monitor tenant database sizes
watch -n 60 './docker/scripts/tenant-manager.sh list'

# Check tenant queue depths
docker exec acme-redis redis-cli --scan \
    --pattern "queues:tenant:*" | \
    xargs -I {} sh -c 'echo -n "{}: "; redis-cli LLEN {}'
```

## Best Practices

1. **Tenant ID Naming**
   - Use lowercase alphanumeric with hyphens
   - Keep under 32 characters
   - Use meaningful identifiers

2. **Resource Limits**
   - Set per-tenant storage quotas
   - Implement rate limiting
   - Monitor resource usage

3. **Backup Strategy**
   - Daily automated backups
   - Retain backups for 30 days
   - Test restore procedures regularly

4. **Security**
   - Regular security audits
   - Tenant data encryption at rest
   - Implement tenant-aware logging

5. **Scaling**
   - Monitor database sizes
   - Plan for horizontal scaling
   - Use read replicas for large tenants

---

**Last Updated**: 2025-01-17
**Version**: 1.0.0

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved