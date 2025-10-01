# Deployment Guide

## Overview

The ACME Corp CSR Platform supports multiple deployment strategies designed for enterprise-grade scalability and reliability. This guide covers Docker-based deployments, production configurations, and infrastructure setup for high-availability environments.

## Deployment Options

### Quick Environment Overview

| Environment | Purpose | Configuration | Access |
|------------|---------|---------------|---------|
| Local Development | Development work | `.env` + `php artisan serve` | `http://localhost:8000` |
| Docker Development | Container-based dev | `.env.docker` + Docker Compose | `http://localhost:8000` |
| Docker Staging | Pre-production testing | Docker Compose + staging config | Custom domain |
| Docker Production | Live environment | Docker Compose + production overlay | Custom domain |

### Quick Environment Switching

```bash
# Docker Development Environment (Recommended)
docker-compose --env-file .env.docker up -d

# Docker Production Environment
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Local Development Environment (uses .env automatically)
php artisan serve

# Stop Docker containers
docker-compose down
```

## Docker Deployment (Recommended)

### Prerequisites

- Docker 20.x or higher
- Docker Compose 2.x or higher
- 4GB+ RAM available for containers
- 10GB+ disk space for images and volumes

### Development Environment Setup

#### 1. Initial Setup

```bash
# Clone repository
git clone https://github.com/go2digital-org/acme-corp-optimy.git
cd acme-corp-optimy

# Copy environment files
cp .env.docker.example .env.docker
```

#### 2. Environment Configuration

Edit `.env.docker` for Docker services:

```env
# Application
APP_NAME="ACME Corp CSR Platform"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

# Database (Docker service names)
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=acme_corp_csr
DB_USERNAME=acme
DB_PASSWORD=secret

# Redis (Docker service name)
REDIS_HOST=redis
REDIS_PORT=6379

# Meilisearch (Docker service name)
MEILISEARCH_HOST=http://meilisearch:7700
MEILISEARCH_KEY=masterKey

# Mail (Docker service name)
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025

# Ports for external access
APP_PORT=8000
MYSQL_PORT=3306
REDIS_PORT=6379
MEILISEARCH_PORT=7700
MAILPIT_PORT=8025
```

#### 3. Start Development Environment

```bash
# Start all services
docker-compose --env-file .env.docker up -d

# Install dependencies
docker-compose exec app composer install
docker-compose exec app npm install && npm run build

# Run migrations and seeders
docker-compose exec app php artisan migrate --seed

# Configure search indexes
docker-compose exec app php artisan scout:sync-index-settings
docker-compose exec app php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'
```

#### 4. Verify Installation

Access points:
- **Application**: `http://localhost:8000`
- **API Documentation**: `http://localhost:8000/api`
- **Mailpit (Email testing)**: `http://localhost:8025`
- **Meilisearch**: `http://localhost:7700`

### Production Environment Setup

#### 1. Production Configuration

Create `docker-compose.prod.yml` overlay:

```yaml
version: '3.8'

services:
  app:
    environment:
      - APP_ENV=production
      - APP_DEBUG=false
      - LOG_LEVEL=warning
    volumes:
      - ./storage/logs:/var/www/html/storage/logs
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  mysql:
    volumes:
      - mysql_data_prod:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  redis:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  meilisearch:
    volumes:
      - meilisearch_data_prod:/meili_data
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G

  queue-worker:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

volumes:
  mysql_data_prod:
    driver: local
  meilisearch_data_prod:
    driver: local
```

#### 2. Production Environment Variables

Create `.env.production`:

```env
# Application
APP_NAME="ACME Corp CSR Platform"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://yourdomain.com

# Security
APP_KEY=base64:YOUR_32_CHARACTER_SECRET_KEY

# Database
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=acme_corp_csr_prod
DB_USERNAME=acme_prod
DB_PASSWORD=SECURE_DATABASE_PASSWORD
DB_ROOT_PASSWORD=SECURE_ROOT_PASSWORD

# Redis
REDIS_HOST=redis
REDIS_PASSWORD=SECURE_REDIS_PASSWORD

# Meilisearch
MEILISEARCH_HOST=http://meilisearch:7700
MEILISEARCH_KEY=SECURE_MASTER_KEY

# Mail Configuration
MAIL_MAILER=smtp
MAIL_HOST=your-smtp-server.com
MAIL_PORT=587
MAIL_USERNAME=noreply@yourdomain.com
MAIL_PASSWORD=SECURE_MAIL_PASSWORD
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@yourdomain.com
MAIL_FROM_NAME="${APP_NAME}"

# Queue
QUEUE_CONNECTION=redis
BROADCAST_DRIVER=redis
CACHE_DRIVER=redis
SESSION_DRIVER=redis

# External Services
PAYMENT_PROVIDER=mollie
MOLLIE_KEY=PRODUCTION_MOLLIE_KEY

# Logging
LOG_CHANNEL=stack
LOG_LEVEL=error
LOG_SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK

# Monitoring
SENTRY_LARAVEL_DSN=https://your-sentry-dsn
```

#### 3. Deploy Production Environment

```bash
# Build production images
docker-compose -f docker-compose.yml -f docker-compose.prod.yml build

# Start production services
docker-compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.production up -d

# Run production deployment script
docker-compose exec app php artisan deploy:post
```

### SSL/HTTPS Configuration

#### Using Nginx Proxy Manager (Recommended)

1. **Install Nginx Proxy Manager**:
```yaml
# Add to docker-compose.yml
nginx-proxy-manager:
  image: 'jc21/nginx-proxy-manager:latest'
  restart: unless-stopped
  ports:
    - '80:80'
    - '81:81'
    - '443:443'
  volumes:
    - ./nginx-data:/data
    - ./nginx-letsencrypt:/etc/letsencrypt
```

2. **Configure SSL Certificate**:
   - Access Nginx Proxy Manager at `http://localhost:81`
   - Add proxy host pointing to your application container
   - Enable SSL with Let's Encrypt certificate

#### Using Traefik

```yaml
# traefik.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.9
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=admin@yourdomain.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

  app:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(\`yourdomain.com\`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=myresolver"
```

## Traditional Server Deployment

### Prerequisites

- **PHP 8.4** with required extensions
- **MySQL 8.0** or higher
- **Redis 7.0** or higher
- **Nginx** or **Apache** web server
- **Node.js 18+** for asset compilation
- **Composer 2.0+**
- **Supervisor** for queue workers

### Server Setup

#### 1. Install Dependencies

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install php8.4 php8.4-fpm php8.4-mysql php8.4-redis php8.4-xml php8.4-curl php8.4-mbstring php8.4-zip php8.4-bcmath php8.4-intl

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs

# Install Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# Install MySQL and Redis
sudo apt install mysql-server redis-server
```

#### 2. Web Server Configuration

**Nginx Configuration**:

```nginx
# /etc/nginx/sites-available/acme-corp-csr
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/acme-corp-csr/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Enable compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Security headers
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "strict-origin-when-cross-origin";
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';";
}
```

#### 3. Application Deployment

```bash
# Create application directory
sudo mkdir -p /var/www/acme-corp-csr
cd /var/www/acme-corp-csr

# Clone repository
git clone https://github.com/go2digital-org/acme-corp-optimy.git .

# Install dependencies
composer install --no-dev --optimize-autoloader
npm install && npm run build

# Set permissions
sudo chown -R www-data:www-data /var/www/acme-corp-csr
sudo chmod -R 755 /var/www/acme-corp-csr
sudo chmod -R 775 storage bootstrap/cache

# Create environment file
cp .env.example .env

# Generate application key
php artisan key:generate

# Run migrations
php artisan migrate --force

# Cache configuration
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

#### 4. Queue Worker Configuration

**Supervisor Configuration**:

```ini
# /etc/supervisor/conf.d/acme-corp-csr-worker.conf
[program:acme-corp-csr-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/acme-corp-csr/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/www/acme-corp-csr/storage/logs/worker.log
stopwaitsecs=3600
```

```bash
# Start supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start acme-corp-csr-worker:*
```

#### 5. Scheduler Configuration

```bash
# Add to crontab
sudo crontab -e

# Add this line:
* * * * * cd /var/www/acme-corp-csr && php artisan schedule:run >> /dev/null 2>&1
```

## Automated Deployment

### Deployment Script

Create `deploy.sh` for automated deployments:

```bash
#!/bin/bash

set -e

# Configuration
PROJECT_DIR="/var/www/acme-corp-csr"
BACKUP_DIR="/var/backups/acme-corp-csr"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "Starting deployment at $(date)"

# Create backup
echo "Creating backup..."
mkdir -p $BACKUP_DIR
mysqldump acme_corp_csr > $BACKUP_DIR/database_$TIMESTAMP.sql
tar -czf $BACKUP_DIR/files_$TIMESTAMP.tar.gz -C $PROJECT_DIR .

# Update code
echo "Updating code..."
cd $PROJECT_DIR
git fetch origin
git reset --hard origin/main

# Install dependencies
echo "Installing dependencies..."
composer install --no-dev --optimize-autoloader
npm ci && npm run build

# Run migrations
echo "Running migrations..."
php artisan migrate --force

# Clear and cache
echo "Clearing caches..."
php artisan config:clear
php artisan view:clear
php artisan cache:clear

echo "Caching configuration..."
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Restart services
echo "Restarting services..."
sudo supervisorctl restart acme-corp-csr-worker:*
sudo systemctl reload nginx

# Run post-deployment tasks
echo "Running post-deployment tasks..."
php artisan deploy:post

echo "Deployment completed successfully at $(date)"
```

### GitHub Actions Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: 8.4
        extensions: mbstring, xml, ctype, iconv, intl, pdo_mysql

    - name: Install dependencies
      run: composer install --no-dev --optimize-autoloader

    - name: Build assets
      run: |
        npm ci
        npm run build

    - name: Deploy to server
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /var/www/acme-corp-csr
          git pull origin main
          composer install --no-dev --optimize-autoloader
          npm ci && npm run build
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          sudo supervisorctl restart acme-corp-csr-worker:*
          sudo systemctl reload nginx
```

## Environment-Specific Commands

### Post-Deployment Setup

The platform includes a comprehensive post-deployment command:

```bash
# Complete post-deployment setup
php artisan deploy:post
```

This command handles:
- Database migrations
- Cache optimization
- Search index configuration
- Queue worker restart
- Permission fixes

### Individual Setup Commands

```bash
# Database setup
php artisan migrate --force
php artisan db:seed --class=ProductionSeeder

# Cache optimization
php artisan optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Search configuration
php artisan meilisearch:configure
php artisan scout:sync-index-settings
php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'

# Queue management
php artisan queue:restart
php artisan horizon:terminate

# Permission fixes
chmod -R 755 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

## Monitoring and Health Checks

### Application Health Checks

```bash
# Health check endpoint
curl -f http://localhost:8000/health || exit 1

# Database connection check
php artisan tinker --execute="DB::connection()->getPdo();"

# Redis connection check
php artisan tinker --execute="Redis::ping();"

# Meilisearch connection check
curl -f http://localhost:7700/health
```

### Log Monitoring

```bash
# Monitor application logs
tail -f storage/logs/laravel.log

# Monitor web server logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Monitor queue worker logs
tail -f storage/logs/worker.log
```

### Performance Monitoring

```bash
# Monitor Docker container resources
docker stats

# Monitor system resources
htop
iostat -x 1
netstat -tuln
```

## Backup and Recovery

### Database Backup

```bash
# Create database backup
mysqldump -u username -p acme_corp_csr > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore database backup
mysql -u username -p acme_corp_csr < backup_20250118_120000.sql
```

### File Backup

```bash
# Create file backup
tar -czf backup_files_$(date +%Y%m%d_%H%M%S).tar.gz \
    --exclude='storage/logs' \
    --exclude='node_modules' \
    --exclude='vendor' \
    .

# Restore file backup
tar -xzf backup_files_20250118_120000.tar.gz
```

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/var/backups/acme-corp-csr"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
KEEP_DAYS=30

mkdir -p $BACKUP_DIR

# Database backup
mysqldump acme_corp_csr | gzip > $BACKUP_DIR/db_$TIMESTAMP.sql.gz

# Files backup
tar -czf $BACKUP_DIR/files_$TIMESTAMP.tar.gz \
    --exclude='storage/logs' \
    --exclude='node_modules' \
    /var/www/acme-corp-csr

# Cleanup old backups
find $BACKUP_DIR -name "*.gz" -mtime +$KEEP_DAYS -delete

echo "Backup completed: $TIMESTAMP"
```

## Troubleshooting

### Common Issues

#### Container Issues

```bash
# Check container status
docker-compose ps

# View container logs
docker-compose logs -f app
docker-compose logs mysql
docker-compose logs redis

# Restart specific service
docker-compose restart app

# Rebuild containers
docker-compose build --no-cache
```

#### Database Issues

```bash
# Check database connection
docker-compose exec mysql mysql -u acme -p

# Reset database
docker-compose exec app php artisan migrate:fresh --seed

# Check migration status
docker-compose exec app php artisan migrate:status
```

#### Permission Issues

```bash
# Fix Laravel permissions
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache

# Fix Docker permissions
sudo chown -R $USER:$USER .
```

#### Search Issues

```bash
# Reset search indexes
php artisan scout:flush 'Modules\Campaign\Domain\Model\Campaign'
php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'

# Check Meilisearch status
curl http://localhost:7700/health
```

### Performance Issues

```bash
# Check container resource usage
docker stats

# Optimize caches
php artisan optimize:clear
php artisan optimize

# Check queue status
php artisan queue:monitor redis
```

## Security Considerations

### Environment Security

- Use strong passwords for all services
- Keep environment files secure and out of version control
- Regularly update dependencies
- Use HTTPS in production
- Configure proper firewall rules

### Application Security

```bash
# Security scan
php artisan enlightn

# Update dependencies
composer update
npm audit fix

# Check for known vulnerabilities
composer audit
```

### Infrastructure Security

- Regular server updates
- SSH key authentication
- Fail2ban for intrusion prevention
- Regular security audits
- Backup encryption

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved