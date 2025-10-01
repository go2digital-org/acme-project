# Go2digit.al ACME Corp CSR Platform - Documentation

## Overview

This documentation provides comprehensive technical guidance for the Go2digit.al ACME Corp CSR Platform, a multi-tenant corporate social responsibility platform built using hexagonal architecture with PHP/Laravel and Vue.js.

## Navigation

- [Architecture & Design](#architecture--design)
- [Development](#development)
- [Infrastructure & Operations](#infrastructure--operations)
- [Quality Assurance](#quality-assurance)
- [API Documentation](#api-documentation)
- [Module Guides](#module-guides)
- [Docker & Containerization](#docker--containerization)
- [FAQ](#faq)

---

## Architecture & Design

Comprehensive guides covering the platform's architectural decisions and design patterns.

### Core Architecture
- **[Hexagonal Architecture](architecture/hexagonal-architecture.md)** - Domain-driven design implementation details
- **[Module Structure Guide](architecture/module-structure.md)** - Hexagonal architecture module organization and boundaries
- **[Module Interactions](architecture/module-interactions.md)** - Cross-module communication patterns and interfaces

### Platform Design
- **[API Platform Design](architecture/api-platform-design.md)** - RESTful API design principles and standards
- **[API Platform](architecture/api-platform.md)** - Platform API architecture and implementation
- **[Payment Architecture](architecture/payment-architecture.md)** - Payment processing system design and integration
- **[Internationalization](architecture/internationalization.md)** - Multi-language support and localization strategy

---

## Development

Essential guides for developers working on the platform.

### Getting Started
- **[Developer Guide](development/developer-guide.md)** - Complete development workflow and best practices
- **[Getting Started](development/getting-started.md)** - Environment setup and initial development steps
- **[Hex Commands](development/hex-commands.md)** - Custom Artisan commands for hexagonal architecture

### Development Processes
- **[CI/CD Pipeline](development/ci-cd-pipeline.md)** - Continuous integration and deployment workflows
- **[Localization](development/translation.md)** - Multi-language implementation and management
- **[Code Quality & Linting](development/code-quality.md)** - Code standards, static analysis, and quality metrics
- **[Testing & Coverage](development/linting-testing.md)** - Testing strategies and coverage analysis

---

## Infrastructure & Operations

Production deployment, server management, and operational procedures.

### Deployment & Configuration
- **[Deployment Guide](infrastructure/deployment.md)** - Production deployment procedures and requirements
- **[Infrastructure Setup](infrastructure/readme.md)** - Complete infrastructure setup and management
- **[Environment Audit](infrastructure/env-audit.md)** - Environment configuration validation and security

### Server Setup & Management
- **[Staging Server Setup](infrastructure/staging-server-setup.md)** - Staging environment configuration and deployment
- **[Tenancy Setup](infrastructure/tenancy-setup.md)** - Multi-tenant architecture configuration
- **[GitHub Runner Setup](infrastructure/github-runner-setup.md)** - Self-hosted CI/CD runner configuration

### Caching & Performance
- **[Redis Cache](infrastructure/redis-cache.md)** - Redis caching implementation and configuration
- **[Redis Cache Usage](infrastructure/redis-cache-usage.md)** - Cache usage patterns and optimization strategies
- **[Redis Queue](infrastructure/redis-queue.md)** - Queue management and background job processing
- **[Dashboard Cache Optimization](infrastructure/dashboard-cache-optimization.md)** - Dashboard performance optimization techniques

### Queue & Worker System
- **[Queue & Worker System](infrastructure/queue-worker-system.md)** - Comprehensive queue system guide with 8 specialized queues
- **[Worker Deployment](infrastructure/worker-deployment.md)** - Worker deployment, scaling, and Docker/K8s configurations
- **[Job Monitoring](infrastructure/job-monitoring.md)** - Real-time monitoring, alerting, and troubleshooting guide

---

## Quality Assurance

Code quality tools, linting configuration, and testing strategies.

### Code Quality Tools
- **[Code Quality & Linting](linting/linting.md)** - Code style enforcement and linting tool configuration
- **[Static Analysis](linting/grumphp-optimization.md)** - Git hooks and automated quality checks

---

## API Documentation

Comprehensive API reference and integration guides.

### API Reference
- **[OpenAPI Documentation](api/openapi-documentation.md)** - Complete API specification and endpoint documentation

---

## Module Guides

Module-specific implementation guides and usage documentation.

### Specialized Guides
- **[Currency Usage](modules/currency-usage.md)** - Multi-currency support implementation and usage
- **[Notification Broadcasting](modules/notification-broadcasting.md)** - Real-time notification system and broadcasting

---

## Docker & Containerization

Container orchestration, deployment, and development environment setup.

### Container Configuration
- **[Docker Deployment](docker/docker-deployment.md)** - Docker-based deployment and orchestration
- **[FrankenPHP](docker/frankenphp.md)** - Modern PHP application server configuration
- **[Supervisor](docker/supervisor.md)** - Process management and queue worker supervision

---

## Quick Reference

### Environment Setup
```bash
# Docker Environment (Recommended)
docker-compose --env-file .env.docker up -d

# Local Development (uses .env automatically)
php artisan serve
```

### Testing
```bash
# Run all tests
./vendor/bin/pest

# Run specific test suites
./vendor/bin/pest --testsuite=Unit
./vendor/bin/pest --testsuite=Integration
./vendor/bin/pest --testsuite=Feature
```

### Quality Checks
```bash
# Code style
./vendor/bin/pint

# Static analysis
./vendor/bin/phpstan analyse

# All quality checks
./vendor/bin/grumphp run
```

---

## FAQ

### Why Hexagonal Architecture?

**Q: Why did we choose hexagonal architecture for this platform?**

A: Hexagonal architecture provides several critical benefits for enterprise applications:

- **Testability**: Business logic is isolated from external dependencies, making unit testing straightforward
- **Flexibility**: Easy to swap infrastructure components (databases, APIs, frameworks) without affecting business logic
- **Scalability**: Clear separation of concerns allows teams to work independently on different layers
- **Maintainability**: Domain logic remains pure and framework-independent, reducing technical debt
- **Enterprise Readiness**: Supports complex business rules and compliance requirements effectively

### Why CQRS Pattern?

**Q: What benefits does CQRS bring to our CSR platform?**

A: Command Query Responsibility Segregation offers:

- **Performance**: Optimized read and write operations for different use cases
- **Analytics**: Separate read models enable powerful reporting without affecting transactional performance
- **Security**: Fine-grained access control between commands and queries
- **Scalability**: Read and write sides can be scaled independently based on usage patterns

### Module Organization

**Q: How do we decide what belongs in each module?**

A: Our modules follow Domain-Driven Design principles:

- **Campaign Module**: All campaign lifecycle management
- **Donation Module**: Payment processing and donation handling
- **User Module**: Authentication, authorization, and user management
- **Organization Module**: Multi-tenant organization features
- **Notification Module**: Communication and alert systems

Each module has clear boundaries and communicates through well-defined interfaces.

---

## Contributing

For contribution guidelines and development workflows, see the [Developer Guide](development/developer-guide.md).

## Support

For technical support and questions, please refer to the relevant documentation sections above or contact the development team.

---

**Developed and Maintained by Go2digit.al**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved