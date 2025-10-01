<div align="center">

# ACME Corp CSR Platform

[![CI Pipeline](https://github.com/go2digital-org/acme-project/actions/workflows/ci.yml/badge.svg)](https://github.com/go2digital-org/acme-project/actions/workflows/ci.yml)
[![Laravel Version](https://img.shields.io/badge/Laravel-12.x-red.svg)](https://laravel.com)
[![API Platform](https://img.shields.io/badge/API%20Platform-4.x-blue.svg)](https://api-platform.com)
[![PHPStan Level](https://img.shields.io/badge/PHPStan-Level%208-brightgreen.svg)](https://phpstan.org)
[![Security Check](https://img.shields.io/badge/Security-Enlightn%20Enabled-blue.svg)](https://www.laravel-enlightn.com/docs/security/security-analyzer.html)
[![Code Quality](https://img.shields.io/badge/Code%20Quality-Pint%20%E2%80%A2%20PHPStan%20%E2%80%A2%20Rector%20%E2%80%A2%20Deptrac-0ea5e9.svg)](#quality)
[![Tests](https://img.shields.io/badge/Tests-3,934%20tests%20|%20100%25%20pass%20|%2012,828%20assertions-brightgreen.svg)](#testing)
[![Architecture](https://img.shields.io/badge/Architecture-Hexagonal%20%E2%80%A2%20DDD%20%E2%80%A2%20CQRS-purple.svg)](#architecture)

  <img src="https://go2digit.al/storage/media/a58e6e4f-04bc-4e84-86fb-c18bd0f15e03.png" alt="ACME Corp CSR Platform Banner" width="600">
</div>

Enterprise-grade Corporate Social Responsibility platform engineered with Laravel 12, API Platform 4.x, Hexagonal Architecture, and Domain-Driven Design principles. Features pure API-first architecture with CQRS pattern, designed to handle 20,000+ concurrent users across global operations.

## Screenshots

<div align="center">

### Dashboard & Analytics
<img src="docs/screenshots/dashboard-overview.png" alt="Dashboard Overview" width="800">

*Real-time donation tracking, campaign analytics, and impact metrics*

### Campaign Management
<img src="docs/screenshots/my-campaigns.png" alt="Campaign Management" width="800">

*Create, manage, and track fundraising campaigns with advanced filtering*

### Donation Processing
<img src="docs/screenshots/donation-processing.png" alt="Donation Processing" width="800">

*Secure multi-gateway payment processing with real-time status updates*

### More Screenshots
[View all screenshots →](docs/screenshots/)

</div>

## Tech Stack

### Core Technologies
- Laravel 12.x with PHP 8.4
- API Platform 4.x for pure API-first architecture
- MySQL 8.0 with Redis 7.x caching
- Meilisearch for full-text search
- FrankenPHP with HTTP/3 support

### Development & Quality
- Pest 3.x testing framework with 100% pass rate
- PHPStan Level 8 static analysis
- Laravel Pint, Rector, Deptrac for code quality
- GrumPHP git hooks for automated quality checks

## Quick Start

### Docker Setup (Recommended)
```bash
# Clone and start all services
git clone https://github.com/go2digital-org/acme-corp-optimy.git
cd acme-corp-optimy
docker-compose --env-file .env.docker up -d

# Install dependencies and setup database
docker-compose exec app composer install
docker-compose exec app npm install && npm run build
docker-compose exec app php artisan migrate --seed
docker-compose exec app php artisan scout:sync-index-settings
```

**Access**: http://localhost:8000 | **API Docs**: http://localhost:8000/api

### Local Development
```bash
composer install && npm install
php artisan migrate --seed
php artisan serve
```

## Documentation

### Architecture & Design
- **[Hexagonal Architecture](docs/architecture/hexagonal-architecture.md)** - Clean separation of Domain, Application, and Infrastructure layers
- **[Module Structure Guide](docs/architecture/module-structure.md)** - Standard module organization and boundaries
- **[Module Interactions](docs/architecture/module-interactions.md)** - Inter-module communication patterns
- **[API Platform Design](docs/architecture/api-platform-design.md)** - Pure API-first architecture with zero traditional routes
- **[CQRS Patterns](docs/architecture/cqrs-patterns.md)** - Command Query Responsibility Segregation implementation
- **[Payment Architecture](docs/architecture/payment-architecture.md)** - Secure payment processing design
- **[Internationalization](docs/architecture/internationalization.md)** - Multi-language support strategy

### Development Guides
- **[Developer Guide](docs/development/developer-guide.md)** - Complete development workflow and standards
- **[Getting Started](docs/development/getting-started.md)** - Local development environment setup
- **[Hex Commands](docs/development/hex-commands.md)** - Hexagonal architecture scaffolding commands
- **[Code Quality](docs/development/code-quality.md)** - PHPStan, Pint, Rector, and Deptrac configuration
- **[Linting & Testing](docs/development/linting-testing.md)** - Testing strategies and quality assurance
- **[Translation](docs/development/translation.md)** - Localization and translation management
- **[CI/CD Pipeline](docs/development/ci-cd-pipeline.md)** - Automated testing and deployment pipeline

### Infrastructure & Operations
- **[Deployment Guide](docs/infrastructure/deployment.md)** - Production deployment procedures
- **[Queue Worker System](docs/infrastructure/queue-worker-system.md)** - Background job processing architecture
- **[Redis Cache Usage](docs/infrastructure/redis-cache-usage.md)** - Cache strategy and optimization
- **[Meilisearch Troubleshooting](docs/infrastructure/meilisearch-troubleshooting.md)** - Search engine configuration and debugging
- **[Job Monitoring](docs/infrastructure/job-monitoring.md)** - Queue and task monitoring
- **[Staging Server Setup](docs/infrastructure/staging-server-setup.md)** - Staging environment configuration
- **[GitHub Runner Setup](docs/infrastructure/github-runner-setup.md)** - CI/CD runner configuration

### Docker & Containerization
- **[Docker Deployment](docs/docker/docker-deployment.md)** - Container orchestration and deployment
- **[FrankenPHP Configuration](docs/docker/frankenphp.md)** - High-performance PHP server setup
- **[Multi-Tenancy Guide](docs/docker/multi-tenancy-guide.md)** - Multi-tenant deployment strategies
- **[Supervisor Configuration](docs/docker/supervisor.md)** - Process management in containers

### API Documentation
- **[API Documentation](docs/api/api-documentation.md)** - Comprehensive API reference
- **[OpenAPI Documentation](docs/api/openapi-documentation.md)** - Auto-generated API specifications
- **[API Endpoints](docs/api/endpoints.md)** - Available endpoints and usage

### Module Guides
- **[Module Overview](docs/modules/module-overview.md)** - Available modules and their responsibilities
- **[Currency Usage](docs/modules/currency-usage.md)** - Multi-currency support implementation
- **[Notification Broadcasting](docs/modules/notification-broadcasting.md)** - Real-time notification system

### Decision Records (ADRs)
- **[001 - Hexagonal Architecture](docs/adr/001-hexagonal-architecture.md)** - Architecture decision rationale
- **[002 - API Platform](docs/adr/002-api-platform.md)** - API-first architecture adoption
- **[003 - CQRS Pattern](docs/adr/003-cqrs-pattern.md)** - Command Query separation strategy
- **[004 - Domain-Driven Design](docs/adr/004-domain-driven-design.md)** - DDD implementation approach
- **[005 - Employee to User Migration](docs/adr/005-employee-to-user-migration.md)** - User model consolidation

## Requirements

### System Requirements

- **PHP 8.4** or higher
- **MySQL 8.0** or higher
- **Redis 7.0** or higher
- **Node.js 18+** and NPM/Yarn
- **Composer 2.0+**
- **Meilisearch 1.0+** (for search functionality)
- **Docker & Docker Compose** (for containerized setup)

### PHP Extensions

- BCMath PHP Extension
- Ctype PHP Extension
- cURL PHP Extension
- DOM PHP Extension
- Fileinfo PHP Extension
- JSON PHP Extension
- Mbstring PHP Extension
- OpenSSL PHP Extension
- PCRE PHP Extension
- PDO PHP Extension
- Tokenizer PHP Extension
- XML PHP Extension
- Redis PHP Extension

## Installation

### Option 1: Docker Setup (Recommended)

The project supports both Docker and local development environments with separate configurations.

#### Quick Start with Docker

```bash
# Clone the repository
git clone https://github.com/go2digit-al/acme-corp-optimy.git
cd acme-corp-optimy

# Start all services with Docker Compose
docker-compose --env-file .env.docker up -d

# Install dependencies
docker-compose exec app composer install
docker-compose exec app npm install && npm run build

# Run migrations and seed data
docker-compose exec app php artisan migrate --seed

# Configure search indexes
docker-compose exec app php artisan scout:sync-index-settings
docker-compose exec app php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'
```

**Access URLs:**
- Application: http://app.acme-corp-optimy.orb.local/ (OrbStack) or http://localhost:8000
- API Documentation: http://localhost:8000/api
- Mailpit (Email testing): http://localhost:8025
- Meilisearch: http://localhost:7700

#### Environment Configuration

Docker uses `.env.docker` with service names for inter-container communication:

```env
# Docker Environment (.env.docker)
DB_HOST=mysql
REDIS_HOST=redis
MEILISEARCH_HOST=http://meilisearch:7700
MAIL_HOST=mailpit
```

### Option 2: Local Development

#### Switch to Local Environment

```bash
# Switch to local environment configuration
./scripts/env-local.sh

# Install dependencies
composer install --optimize-autoloader
npm install

# Generate application key
php artisan key:generate
```

#### Local Environment Setup

Configure your `.env` file for local services:

```env
# Local Environment (.env)
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=acme_corp_csr
DB_USERNAME=your_username
DB_PASSWORD=your_password

REDIS_HOST=127.0.0.1
REDIS_PORT=6379

MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=your_master_key
```

#### Database Setup

```bash
# Create database
mysql -u root -p -e "CREATE DATABASE acme_corp_csr CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Run migrations
php artisan migrate

# Seed development data (optional)
php artisan db:seed
```

#### Start Development Server

```bash
# Start Laravel development server
php artisan serve

# In separate terminal, start queue workers
php artisan queue:work redis --queue=high,default,low --tries=3
```

The application will be available at `http://localhost:8000`

### Post-Installation Setup

#### Search Configuration

```bash
# Configure Meilisearch index settings
php artisan meilisearch:configure

# Import search indexes (500k+ campaigns)
php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'

# Monitor indexing progress
php artisan scout:index-monitor 'Modules\Campaign\Domain\Model\Campaign'
```

#### Build Assets

```bash
# Development
npm run dev

# Production
npm run build
```

#### Cache Optimization (Production)

```bash
php artisan optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

## Testing

### Test Statistics
- **3,934 tests** with **100% pass rate**
- **12,828 assertions** across all test suites
- **860+ unit tests** covering pure domain logic
- **200+ integration tests** for infrastructure layer
- **Feature tests** for API Platform endpoints
- **Browser tests** using Pest 4 Browser Plugin
- **Architecture tests** ensuring hexagonal boundaries (0 violations)

### Test Commands
```bash
# Run all tests with parallel execution
./vendor/bin/pest --parallel

# Run by test suite
./vendor/bin/pest --testsuite=Unit          # Domain logic tests
./vendor/bin/pest --testsuite=Integration   # Infrastructure tests
./vendor/bin/pest --testsuite=Feature       # API endpoint tests

# Run with coverage
./vendor/bin/pest --coverage --min=80

# Docker environment
docker-compose exec app ./vendor/bin/pest --parallel
```

### Code Quality
```bash
# Run all quality checks
./vendor/bin/grumphp run

# Individual tools
./vendor/bin/phpstan analyse    # Static analysis (Level 8)
./vendor/bin/pint              # Code style fixing
./vendor/bin/rector process    # Automated refactoring
./vendor/bin/deptrac analyse   # Architecture boundaries
```

## Deployment

### Docker Production
```bash
# Production deployment with all services
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Post-deployment setup
docker-compose exec app php artisan deploy:post
docker-compose exec app php artisan optimize
```

### Manual Deployment
```bash
# Installation and optimization
composer install --no-dev --optimize-autoloader
npm run build
php artisan migrate --force
php artisan meilisearch:configure
php artisan scout:sync-index-settings
php artisan optimize

# Cache optimization
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

### Environment Switching
```bash
# Docker development/staging
docker-compose --env-file .env.docker up -d

# Local development (uses .env automatically)
php artisan serve

# Stop containers
docker-compose down
```

## System Requirements

- PHP 8.4 or higher
- MySQL 8.0 or higher
- Redis 7.0 or higher
- Node.js 18+ and NPM/Yarn
- Composer 2.0+
- Meilisearch 1.0+ for search functionality
- Docker & Docker Compose for containerized setup

See **[Docker Deployment](docs/docker/docker-deployment.md)** for detailed container setup and **[Deployment Guide](docs/infrastructure/deployment.md)** for production deployment procedures.

## Roadmap

#### Frontend Decoupling
- [ ] **Vue.js 3 Headless Client**: Develop a modern Vue.js 3 application consuming the API Platform endpoints for complete frontend-backend separation
- [ ] **React Alternative Client**: Create a React-based client implementation demonstrating framework flexibility through the API-first architecture
- [ ] **API Platform Enhancement**: Extend API Platform integration with GraphQL support, advanced filtering, and real-time subscriptions via Mercure

#### General Improvements
- [ ] **Code Cleanup**: Refactor legacy code sections and remove deprecated methods
- [ ] **Bug Fixes**: Address known issues in campaign management, donation processing, and user authentication flows
- [ ] **Performance Optimization**: Optimize database queries and implement additional caching strategies
- [ ] **Technical Debt**: Resolve accumulated technical debt and improve code maintainability

## Support

For technical support, please contact the development team.

## License

This is proprietary software owned by  Go2Digital. All rights reserved.

---


<div align="center">
  <img src="https://go2digit.al/storage/profile-photos/19SUTKoKhYGA30JGrl5pjzQyQuXIfvAQ42wuRWGR.png" alt="Go2digit.al Logo" width="150">

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved
</div>