# Staging Server Setup and Troubleshooting Guide

## Overview

This document details the complete setup and troubleshooting process for the ACME Corp CSR Platform staging server.

## Final Working Configuration

### Server Stack
- **OS**: Ubuntu Server (Hetzner Cloud)
- **Web Server**: nginx/1.24.0
- **PHP**: PHP 8.4.12 with PHP-FPM
- **Database**: MySQL 8.0
- **SSL**: Let's Encrypt certificates via Certbot
- **Application**: Laravel 11 with Stancl Tenancy

### Key Configuration Files

#### 1. Nginx Configuration (`/etc/nginx/sites-available/staging.yourdomain.com`)

```nginx
server {
    listen 80;
    server_name staging.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name staging.yourdomain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/staging.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/staging.yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /var/www/acme-staging/current/public;
    index index.php index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    # Logging
    access_log /var/log/nginx/staging.acme-corp-access.log;
    error_log /var/log/nginx/staging.acme-corp-error.log;

    # Client configuration
    client_max_body_size 100M;
    client_body_timeout 120s;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_param HTTP_HOST $http_host;
        fastcgi_param SERVER_NAME $server_name;
        fastcgi_param HTTPS on;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

#### 2. Environment Configuration (`.env`)

Key settings for staging environment:

```env
APP_NAME="ACME CSR Platform"
APP_ENV=staging
APP_DEBUG=false
APP_URL=https://staging.yourdomain.com

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=acme_corp_optimy
DB_USERNAME=root
DB_PASSWORD=root

# Cache and Sessions (File-based for staging)
CACHE_STORE=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync

# Multi-Tenancy
CENTRAL_DOMAIN=staging.yourdomain.com
TENANT_IDENTIFICATION=subdomain
```

#### 3. Tenancy Configuration (`config/tenancy.php`)

**Critical Fix**: Add staging domain to central domains array:

```php
'central_domains' => [
    '127.0.0.1',
    'localhost',
    'acme-corp-optimy.test',
    'staging.yourdomain.com',  // <- This line was missing!
],
```

## Issues Encountered and Solutions

### 1. SSL Certificate Issues

**Problem**: Initial SSL certificate only covered `yourdomain.com`, not the staging subdomain.

**Error**: 
```
SSL handshake failed
```

**Solution**:
```bash
certbot certonly --nginx -d staging.yourdomain.com
```

### 2. Tenant Not Found Error

**Problem**: Application was treating `staging.yourdomain.com` as a tenant subdomain instead of the central domain.

**Error**:
```
Tenant not found for subdomain: staging
vendor/laravel/framework/src/Illuminate/Foundation/Application.php#1423
Symfony\Component\HttpKernel\Exception\NotFoundHttpException
```

**Root Cause**: The staging domain was not included in the `central_domains` array in `config/tenancy.php`.

**Solution**: Add `staging.yourdomain.com` to the central domains list:

```php
'central_domains' => [
    '127.0.0.1',
    'localhost',
    'acme-corp-optimy.test',
    'staging.yourdomain.com',
],
```

### 3. Database Connection Issues

**Problem**: Database `acme_corp_optimy` didn't exist and credentials were incorrect.

**Solution**:
```bash
# Create database
mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS acme_corp_optimy;"

# Update .env file
DB_DATABASE=acme_corp_optimy
DB_USERNAME=root
DB_PASSWORD=root

# Run migrations
php artisan migrate --force
```

### 4. Redis Connection Issues

**Problem**: Application configured for Redis but staging environment had connection issues.

**Error**:
```
Call to a member function connect() on null (View: .../navigation.blade.php)
```

**Solution**: Switch to file-based cache and sessions for staging:
```env
CACHE_STORE=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync
```

### 5. FrankenPHP/Octane Routing Issues

**Problem**: FrankenPHP with Laravel Octane was returning 404 for all routes, even though the service was running.

**Symptoms**:
- Service status: Active (running)
- Direct curl to `http://127.0.0.1:8090/`: 404 Not Found
- Laravel logs: "404 HEAD / ....... 0.16 ms"

**Attempted Solutions**:
- FrankenPHP with custom Caddyfile configurations
- RoadRunner server (failed to start)
- Various proxy configurations

**Final Solution**: Reverted to standard nginx + PHP-FPM setup, which provides:
- Better compatibility with Laravel applications
- More predictable routing behavior
- Easier debugging and maintenance
- Production-proven stability

## Deployment Process

### 1. SSL Certificate Setup
```bash
certbot certonly --nginx -d staging.yourdomain.com
```

### 2. Database Setup
```bash
mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS acme_corp_optimy;"
cd /var/www/acme-staging/current
php artisan migrate --force
```

### 3. Configuration Updates
```bash
# Update .env file with correct settings
# Update config/tenancy.php to include staging domain
php artisan config:clear
php artisan cache:clear
```

### 4. Nginx Configuration
```bash
# Create/update nginx site configuration
nginx -t
systemctl reload nginx
```

## Performance Considerations

The current PHP-FPM setup provides:
- **Stability**: Production-proven technology stack
- **Performance**: Adequate for staging workloads
- **Debugging**: Easy to troubleshoot issues
- **Compatibility**: Full Laravel feature support

For production environments requiring higher performance, consider:
- Laravel Octane with Swoole (once routing issues are resolved)
- Horizontal scaling with load balancers
- Database optimization and caching strategies

## Monitoring and Maintenance

### Log Locations
- **nginx Access**: `/var/log/nginx/staging.acme-corp-access.log`
- **nginx Error**: `/var/log/nginx/staging.acme-corp-error.log`
- **Laravel Application**: `/var/www/acme-staging/current/storage/logs/laravel.log`
- **PHP-FPM**: `/var/log/php8.4-fpm.log`

### Health Checks
```bash
# Test site availability
curl -I https://staging.yourdomain.com/

# Check services
systemctl status nginx
systemctl status php8.4-fpm
systemctl status mysql

# Check database connection
php artisan tinker --execute="DB::connection()->getPdo(); echo 'Connected';"
```

## Lessons Learned

1. **Multi-tenancy Configuration**: Always ensure staging/production domains are properly configured in the tenancy package's central domains list.

2. **FrankenPHP Limitations**: While FrankenPHP offers performance benefits, it may have compatibility issues with complex Laravel applications using custom routing or middleware.

3. **Environment Isolation**: Use file-based cache/sessions for staging to avoid Redis dependency issues.

4. **SSL Certificates**: Always create separate certificates for staging subdomains rather than relying on wildcard certificates.

5. **Database Naming**: Ensure consistent database naming across environments to avoid connection issues.

## Troubleshooting Checklist

When encountering issues with the staging server:

1. Check SSL certificate covers the domain
2. Verify database exists and credentials are correct
3. Confirm staging domain is in `central_domains` array
4. Ensure cache/session storage is properly configured
5. Check nginx configuration syntax with `nginx -t`
6. Verify PHP-FPM service is running
7. Review application logs for specific errors
8. Test database connectivity
9. Clear Laravel caches after configuration changes

---

**Status**: Staging server fully operational at `https://staging.yourdomain.com/`

**Last Updated**: September 16, 2025

**Configuration**: nginx + PHP-FPM + MySQL + Let's Encrypt SSL

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved