# ACME Corp CSR Platform - Supervisor Configuration

This directory contains a comprehensive Supervisor configuration that consolidates the ACME CSR platform into a single, unified container. This replaces the previous multi-container setup (app, queue-worker, horizon, scheduler) with a single container managing all processes.

## Overview

### Architecture Consolidation

**Before (Multi-Container):**
- `acme-app` - FrankenPHP web server
- `acme-queue-worker` - Basic queue workers  
- `acme-horizon` - Laravel Horizon monitoring
- `acme-scheduler` - Laravel scheduler

**After (Unified Container):**
- `acme-app` - All processes managed by Supervisor
  - FrankenPHP web server with HTTP/3 support
  - Individual queue workers with priority-based processing
  - Laravel Scheduler for cron job management
  - Priority-based queue workers (8 different worker groups)
  - Health checks and process monitoring

### Benefits

1. **Simplified Deployment**: Single container to manage
2. **Resource Efficiency**: Shared memory and optimized resource usage
3. **Process Management**: Centralized logging and monitoring
4. **Environment Scaling**: Easy scaling per environment
5. **Operational Simplicity**: One container to monitor and debug

## Directory Structure

```
docker/supervisor/
├── supervisord.conf              # Main Supervisor configuration
├── conf.d/                       # Individual process configurations
│   ├── frankenphp.conf          # Web server process
│   ├── horizon.conf             # Queue monitoring
│   ├── scheduler.conf           # Laravel scheduler
│   ├── queues-high.conf         # High priority queues
│   ├── queues-medium.conf       # Medium priority queues
│   └── queues-low.conf          # Low priority queues
├── env/                         # Environment-specific worker scaling
│   ├── production.env           # Production worker counts
│   ├── staging.env              # Staging worker counts
│   └── local.env                # Local development worker counts
├── Dockerfile.unified           # Unified container build
├── docker-compose.unified.yml   # Unified compose configuration
├── start.sh                     # Container startup script
├── supervisor-ctl.sh            # Management script
└── README.md                    # This documentation
```

## Queue Worker Configuration

### Priority Groups

#### High Priority (400-401)
- **Payments**: 2-8 workers, 256MB memory, 5 retries, 180s timeout
- **Notifications**: 1-4 workers, 128MB memory, 3 retries, 120s timeout

#### Medium Priority (500-502)  
- **Exports**: 2 workers, 1024MB memory, 2 retries, 900s timeout
- **Reports**: 2 workers, 512MB memory, 3 retries, 600s timeout
- **Default**: 1-3 workers, 256MB memory, 3 retries, 300s timeout

#### Low Priority (600-603)
- **Bulk**: 1 worker, 1024MB memory, 1 retry, 3600s timeout
- **Maintenance**: 1 worker, 256MB memory, 1 retry, 1800s timeout
- **Cache Warming**: 1-2 workers, 128MB memory, 2 retries, 300s timeout
- **Cache Warming Orchestrator**: 1 worker, 64MB memory, 3 retries, 120s timeout

### Environment-Specific Scaling

#### Production Environment
```bash
Total Workers: 17 processes
Memory Estimate: ~4.5GB for queue workers
High Availability: Full scaling for production load
```

#### Staging Environment  
```bash
Total Workers: 9 processes
Memory Estimate: ~2GB for queue workers
Reduced Capacity: Scaled down for testing
```

#### Local Development
```bash
Total Workers: 8 processes  
Memory Estimate: ~1GB for queue workers
Minimal Setup: Development and testing
```

## Usage

### Quick Start

```bash
# Start in production mode
./docker/supervisor/supervisor-ctl.sh start production

# Start in staging mode
./docker/supervisor/supervisor-ctl.sh start staging

# Start in local development mode
./docker/supervisor/supervisor-ctl.sh start local
```

### Management Commands

```bash
# Show all process status
./docker/supervisor/supervisor-ctl.sh status

# View logs from all processes
./docker/supervisor/supervisor-ctl.sh logs

# View logs from specific process
./docker/supervisor/supervisor-ctl.sh logs queue-payments

# Open shell in container
./docker/supervisor/supervisor-ctl.sh shell

# Check queue worker status
./docker/supervisor/supervisor-ctl.sh workers

# Get Horizon dashboard URL
./docker/supervisor/supervisor-ctl.sh horizon

# Check container health
./docker/supervisor/supervisor-ctl.sh health

# Stop all processes
./docker/supervisor/supervisor-ctl.sh stop
```

### Direct Supervisor Commands

```bash
# Execute commands inside container
docker exec acme-app supervisorctl status
docker exec acme-app supervisorctl restart queue-payments:*
docker exec acme-app supervisorctl stop horizon
docker exec acme-app supervisorctl start horizon
docker exec acme-app supervisorctl tail -f queue-notifications
```

## Process Management

### Automatic Restart Policies

- **FrankenPHP**: Always restart, graceful shutdown (QUIT signal)
- **Horizon**: Always restart, 30s stop timeout for queue cleanup
- **Scheduler**: Always restart, 60s stop timeout
- **Queue Workers**: Always restart, variable timeouts based on job complexity

### Health Checks

- **Container Level**: HTTP health check every 30s
- **Application Level**: Laravel health checks
- **Process Level**: Supervisor process monitoring
- **Queue Level**: Horizon queue monitoring

### Logging

- **Centralized**: All logs to stdout/stderr for container logging
- **Structured**: Process-specific log routing
- **Rotation**: Automatic log rotation (50MB, 10 backups)
- **Real-time**: Live log tailing with supervisor-ctl.sh

## Monitoring & Debugging

### Process Status

```bash
# View all process status
supervisorctl status

# Check specific process
supervisorctl status queue-payments:*

# View process logs
supervisorctl tail -f horizon
```

### Queue Monitoring

```bash
# Laravel Horizon Dashboard
http://localhost/admin/horizon

# Queue statistics
php artisan queue:monitor

# Failed jobs
php artisan horizon:list failed
```

### Resource Monitoring

```bash
# Container resource usage
docker stats acme-app

# Process resource usage inside container
top -p $(pgrep -f "queue:work")
```

## Troubleshooting

### Common Issues

1. **Container won't start**: Check environment variables and dependencies
2. **Queue workers not processing**: Check Redis connection and Horizon status
3. **High memory usage**: Monitor worker memory limits and job complexity
4. **Process crashes**: Check supervisor logs and Laravel logs

### Debug Commands

```bash
# Check container health
./docker/supervisor/supervisor-ctl.sh health

# View all logs
./docker/supervisor/supervisor-ctl.sh logs

# Check specific process
./docker/supervisor/supervisor-ctl.sh logs horizon

# Open shell for debugging
./docker/supervisor/supervisor-ctl.sh shell
```

### Log Locations

- **Supervisor**: `/var/log/supervisor/`
- **Laravel**: `/app/storage/logs/`
- **FrankenPHP**: Container stdout/stderr
- **Queue Workers**: Supervisor process logs

## Migration from Multi-Container

### Steps to Migrate

1. **Stop current setup**:
   ```bash
   docker-compose down
   ```

2. **Backup data** (if needed):
   ```bash
   docker-compose exec mysql mysqldump acme_corp_csr > backup.sql
   ```

3. **Switch to unified setup**:
   ```bash
   cd docker/supervisor
   ./supervisor-ctl.sh start production
   ```

4. **Verify all processes**:
   ```bash
   ./supervisor-ctl.sh status
   ./supervisor-ctl.sh workers
   ./supervisor-ctl.sh horizon
   ```

### Configuration Changes

- **Docker Compose**: Use `docker-compose.unified.yml` instead of root `docker-compose.yml`
- **Environment Variables**: Same variables, different scaling configuration
- **Volumes**: Consolidated into fewer volumes with better caching
- **Networking**: Same network setup, simplified service discovery

## Performance Optimization

### Resource Limits

- **Container**: 6GB memory limit, 4 CPU cores
- **Individual Workers**: Memory limits per queue type
- **PHP OPCache**: Optimized for production workloads
- **Redis**: Memory optimization with LRU eviction

### Scaling Recommendations

#### Production Scaling
- **High Traffic**: Increase payment and notification workers
- **Bulk Operations**: Scale export and bulk workers during maintenance windows
- **Cache Performance**: Increase cache warming workers for better response times

#### Development Scaling
- **Local Testing**: Keep minimal worker counts
- **CI/CD**: Use staging configuration for automated testing
- **Load Testing**: Use production configuration with monitoring

## Security Considerations

- **Process Isolation**: Each process runs as www-data user
- **Resource Limits**: Memory and CPU limits per process
- **Log Security**: No sensitive data in logs
- **Network Security**: Container network isolation
- **File Permissions**: Proper Laravel file permissions

## Future Enhancements

- **Dynamic Scaling**: Auto-scaling based on queue depth
- **Health Metrics**: Prometheus metrics export
- **Log Aggregation**: ELK/EFK stack integration
- **Blue-Green Deployment**: Zero-downtime deployment strategy
- **Multi-Environment**: Environment-specific optimizations

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved