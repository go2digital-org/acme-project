# Multi-Tenancy Setup Guide

## Overview
The ACME Corp CSR Platform now supports multi-tenancy, allowing each organization to have its own isolated environment with separate databases, users, and data.

## Architecture
- **Package**: Stancl/Tenancy v3.9
- **Tenant Model**: Organization (dual-purpose as both organization and tenant)
- **Identification**: Subdomain-based (e.g., `acme.acme-corp-optimy.test`)
- **Isolation**: Separate databases per tenant with prefix `tenant_`
- **Architecture**: All code in `modules/` following hexagonal architecture

## Quick Start

### 1. Environment Setup

#### For Local Development (Valet)
```bash
# Link the project with Valet
valet link acme-corp-optimy

# The LocalValetDriver.php in project root handles subdomain routing
# Access central domain: https://acme-corp-optimy.test
# Access tenant: https://tenant-name.acme-corp-optimy.test
```

#### For Docker Development
```bash
# Standard Docker setup
docker-compose up -d

# With enhanced subdomain routing (using Traefik)
docker-compose -f docker-compose.yml -f docker-compose.tenancy.yml up -d

# Access central domain: http://localhost
# Access tenant: http://tenant-name.localhost
```

### 2. Creating a Tenant via Artisan

```bash
# Interactive tenant creation
php artisan tenant:create

# With options
php artisan tenant:create \
  --name="ACME Foundation" \
  --subdomain=acme \
  --admin-email=admin@acme.org \
  --admin-name="John Doe" \
  --admin-password=[secure-password]

# List all tenants
php artisan tenant:list

# Provision/re-provision a tenant
php artisan tenant:provision acme

# Delete a tenant
php artisan tenant:delete acme --with-database
```

### 3. Creating a Tenant via Filament Admin Panel

1. Login to admin panel: `/admin`
2. Navigate to **Organizations** in the CSR Management section
3. Create a new organization with:
   - Basic organization details
   - **Subdomain** (required for tenant access)
   - Set **Is Active** to true
   - Save the organization

4. After saving, you can:
   - Use bulk action "Provision Tenants" to provision the tenant
   - Or wait for automatic provisioning (if queues are running)
   - Monitor provisioning status in the "Tenant Status" column

5. Once provisioned (status: "active"), access the tenant:
   - Via subdomain: `https://[subdomain].acme-corp-optimy.test`

### 4. Tenant Provisioning Process

When a tenant is provisioned, the system automatically:

1. **Creates tenant database**: `tenant_[id]_[subdomain]`
2. **Runs migrations**: All tenant-specific migrations in modules
3. **Creates super admin**: With provided credentials
4. **Sets up Meilisearch indexes**: Prefixed with tenant ID
5. **Configures storage**: Isolated directories for tenant files

### 5. Testing Tenant Access

```bash
# Test central domain
curl http://acme-corp-optimy.test

# Test tenant subdomain (after creating tenant with subdomain 'acme')
curl http://acme.acme-corp-optimy.test

# Should see different welcome pages for each
```

## Artisan Commands

| Command | Description |
|---------|-------------|
| `tenant:create` | Create a new tenant interactively |
| `tenant:list` | List all tenants with their status |
| `tenant:provision {tenant}` | Provision or re-provision a tenant |
| `tenant:migrate {tenant?}` | Run migrations for specific or all tenants |
| `tenant:delete {tenant}` | Delete a tenant (optionally with database) |

## Tenant Fields in Organization

| Field | Description |
|-------|-------------|
| `subdomain` | Unique subdomain for tenant access |
| `database` | Auto-generated database name |
| `provisioning_status` | Current status: pending, provisioning, active, failed, suspended |
| `provisioning_error` | Error message if provisioning fails |
| `provisioned_at` | Timestamp of successful provisioning |
| `tenant_data` | JSON field for additional tenant metadata |

## Troubleshooting

### Subdomain Not Working

#### Valet
```bash
# Ensure Valet is running
valet status

# Restart Valet
valet restart

# Check if LocalValetDriver.php exists in project root
ls LocalValetDriver.php
```

#### Docker
```bash
# Ensure wildcard SERVER_NAME is set
docker exec acme-app env | grep SERVER_NAME
# Should show: SERVER_NAME=:80, *.localhost:80

# Check Docker logs
docker logs acme-app
```

### Provisioning Failed

1. Check the provisioning error in Filament:
   - Organizations list shows "failed" status
   - Edit organization to see `provisioning_error` field

2. Check Laravel logs:
   ```bash
   tail -f storage/logs/laravel.log
   ```

3. Retry provisioning:
   ```bash
   php artisan tenant:provision [subdomain] --retry
   ```

### Database Issues

```bash
# Check if tenant database exists
mysql -u root -p -e "SHOW DATABASES LIKE 'tenant_%';"

# Manually run tenant migrations
php artisan tenant:migrate [subdomain]
```

## Migration Structure

Tenant migrations are organized by module:

```
modules/
├── User/Infrastructure/Laravel/Migration/Tenant/
│   ├── 2025_01_01_000001_create_tenant_users_table.php
│   └── 2025_01_01_000002_create_tenant_permission_tables.php
├── Campaign/Infrastructure/Laravel/Migration/Tenant/
│   └── 2025_01_01_000003_create_tenant_campaigns_table.php
└── Donation/Infrastructure/Laravel/Migration/Tenant/
    └── 2025_01_01_000004_create_tenant_donations_table.php
```

## Security Considerations

- Each tenant has isolated database with no cross-tenant queries
- Tenant identification happens via middleware before any data access
- Super admin users are created per tenant, not globally
- File storage is isolated per tenant in `storage/app/tenants/[tenant_id]/`

## Next Steps

1. **Configure payment gateways** per tenant
2. **Set up custom domains** for tenants (e.g., `donate.acmefoundation.org`)
3. **Implement tenant billing** for SaaS model
4. **Add tenant resource limits** enforcement
5. **Create tenant-specific themes** and branding

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved