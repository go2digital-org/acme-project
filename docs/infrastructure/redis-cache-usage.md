# Redis Widget Cache Infrastructure

A comprehensive Redis-based caching system designed to support 20,000+ concurrent users with sub-second response times and 99%+ cache hit rates.

## Architecture Overview

The system follows hexagonal architecture patterns with clear separation between domain logic, application services, and infrastructure concerns:

```
modules/Analytics/
├── Application/Service/
│   ├── WidgetCacheService.php              # Main caching orchestrator
│   ├── WidgetDataAggregationService.php    # Optimized data queries
│   ├── WidgetCacheInvalidationService.php  # Smart cache invalidation
│   ├── WidgetCacheMetricsService.php       # Performance monitoring
│   └── WidgetLockManager.php               # Distributed locking
├── Domain/Repository/
│   └── WidgetCacheRepositoryInterface.php  # Cache abstraction
└── Infrastructure/Laravel/
    ├── Repository/
    │   └── WidgetCacheRedisRepository.php   # Redis implementation
    └── Commands/
        ├── WarmWidgetCachesCommand.php      # Cache warming
        ├── WidgetCacheStatusCommand.php     # Status monitoring
        └── WidgetCacheCleanupCommand.php    # Maintenance
```

## Key Features

### 1. Intelligent Caching Strategies
- **Real-time**: 1-minute TTL for live data (recent activities, current stats)
- **Short-lived**: 5-minute TTL for frequently changing data
- **Medium-lived**: 15-minute TTL for moderately dynamic data
- **Long-lived**: 1-hour TTL for historical data
- **Daily**: 24-hour TTL for reports and aggregations

### 2. Distributed Locking
Prevents cache stampedes using Redis-based distributed locks:
```php
$lockAcquired = $lockManager->acquireLock($widgetType, $timeRange, $filters, $timeout);
if ($lockAcquired) {
    // Compute expensive operation
    $lockManager->releaseLock($widgetType, $timeRange, $filters);
}
```

### 3. Refresh-Ahead Pattern
Automatically refreshes cache before expiration to maintain sub-second response times:
```php
// Automatically triggered when cache TTL drops below threshold (80%)
$cacheService->getWidgetData($type, $timeRange, $dataProvider);
```

### 4. Tag-Based Invalidation
Smart invalidation using hierarchical cache tags:
```php
// Invalidate all donation-related widgets
$invalidationService->invalidateByTags(['donation', 'campaign:123']);

// Invalidate specific entity changes
$invalidationService->invalidateForDonation($donation, 'created');
```

## Usage Examples

### Basic Widget Caching

```php
use Modules\Analytics\Application\Service\WidgetCacheService;
use Modules\Analytics\Domain\ValueObject\WidgetType;
use Modules\Analytics\Domain\ValueObject\TimeRange;

// Inject the cache service
public function __construct(
    private WidgetCacheService $cacheService,
    private WidgetDataAggregationService $aggregationService
) {}

// Get cached widget data
$widgetData = $this->cacheService->getWidgetData(
    WidgetType::TOTAL_DONATIONS,
    TimeRange::thisMonth(),
    fn() => $this->aggregationService->aggregateDonationStats(TimeRange::thisMonth())
);
```

### Cache Warming

```php
// Warm specific widgets
$cacheService->warmupWidget(
    WidgetType::CAMPAIGN_PERFORMANCE,
    TimeRange::thisWeek(),
    fn() => $aggregationService->aggregateCampaignStats(TimeRange::thisWeek())
);
```

### Manual Invalidation

```php
// Invalidate specific widget
$cacheService->invalidateWidget(WidgetType::DONATION_TRENDS, TimeRange::today());

// Invalidate by tags
$invalidationService->invalidateByTags(['campaign', 'donation']);

// Smart invalidation based on data changes
$invalidationService->smartInvalidation([
    'donation_created' => [
        'campaign_id' => 123,
        'employee_id' => 456,
        'amount' => 100.0
    ]
]);
```

## Console Commands

### Cache Warming
```bash
# Warm all widget caches
php artisan analytics:warm-cache

# Warm specific widget types
php artisan analytics:warm-cache --type=total_donations --type=campaign_performance

# Warm specific time ranges
php artisan analytics:warm-cache --time-range=today --time-range=this_week

# Run in parallel for faster execution
php artisan analytics:warm-cache --parallel

# Force refresh even if cache exists
php artisan analytics:warm-cache --force
```

### Cache Status Monitoring
```bash
# Basic status overview
php artisan analytics:cache-status

# Detailed metrics
php artisan analytics:cache-status --detailed

# Performance alerts only
php artisan analytics:cache-status --alerts

# Widget-specific status
php artisan analytics:cache-status --widget=total_donations --widget=campaign_performance

# JSON output for monitoring systems
php artisan analytics:cache-status --json
```

### Cache Cleanup
```bash
# Clean expired locks only
php artisan analytics:cache-cleanup --locks

# Clean expired cache entries
php artisan analytics:cache-cleanup --expired-only

# Dry run to see what would be cleaned
php artisan analytics:cache-cleanup --dry-run

# Force cleanup all caches (use with caution)
php artisan analytics:cache-cleanup --force
```

## Configuration

Environment variables for fine-tuning performance:

```env
# Cache TTL Settings
WIDGET_CACHE_DEFAULT_TTL=900
WIDGET_CACHE_REALTIME_TTL=60
WIDGET_CACHE_SHORT_TTL=300
WIDGET_CACHE_MEDIUM_TTL=900
WIDGET_CACHE_LONG_TTL=3600

# Refresh-Ahead Configuration
WIDGET_CACHE_REFRESH_AHEAD=true
WIDGET_CACHE_REFRESH_THRESHOLD=0.8

# Distributed Locking
WIDGET_CACHE_LOCKING_ENABLED=true
WIDGET_CACHE_LOCK_TIMEOUT=30
WIDGET_CACHE_LOCK_RETRIES=3

# Performance Targets
WIDGET_CACHE_MAX_CONCURRENT_USERS=20000
WIDGET_CACHE_TARGET_RESPONSE_TIME=200
WIDGET_CACHE_TARGET_HIT_RATE=99.0

# Monitoring Thresholds
WIDGET_CACHE_MIN_HIT_RATE=90.0
WIDGET_CACHE_MAX_RESPONSE_TIME=500
WIDGET_CACHE_MAX_ERROR_RATE=1.0

# Redis Database Mapping
REDIS_CACHE_DB=1          # Primary cache store
REDIS_QUEUE_DB=2          # Queue system
REDIS_SESSION_DB=3        # Session storage
REDIS_DEFAULT_DB=0        # Default operations
```

## Performance Monitoring

### Metrics Available
- Cache hit/miss rates per widget type
- Average response times
- Lock acquisition success rates
- Memory usage and Redis statistics
- Historical performance trends

### Health Score Calculation
The system provides a 0-100 health score based on:
- **Hit Rate** (50% weight): Higher is better
- **Response Time** (30% weight): Lower is better  
- **Error Rate** (20% weight): Lower is better

### Automated Alerts
The system generates alerts when thresholds are exceeded:
- Hit rate below 90%
- Response time above 500ms
- Redis memory usage above 80%
- High lock failure rates

## Integration with Domain Events

The cache system automatically responds to domain events:

```php
// Campaign events trigger cache invalidation
event(new CampaignCreatedEvent($campaign));
// Automatically invalidates: campaign_performance, organization_stats

// Donation events trigger smart invalidation
event(new DonationCompletedEvent($donation));
// Automatically invalidates: total_donations, donation_trends, top_donors
```

## Production Deployment Checklist

### Redis Configuration
- [ ] Configure Redis with appropriate memory limit
- [ ] Enable Redis persistence (RDB + AOF)
- [ ] Set up Redis clustering for high availability
- [ ] Configure eviction policy: `allkeys-lru`

### Application Configuration
- [ ] Set appropriate TTL values for your use case
- [ ] Enable distributed locking in multi-server setup
- [ ] Configure cache warming schedule
- [ ] Set up monitoring and alerting

### Performance Optimization
- [ ] Monitor hit rates and adjust TTL accordingly
- [ ] Profile widget computation times
- [ ] Optimize database queries in aggregation service
- [ ] Set up proper Redis connection pooling

### Monitoring Setup
- [ ] Integrate with monitoring system (DataDog, New Relic, etc.)
- [ ] Set up dashboard for cache metrics
- [ ] Configure alerts for performance thresholds
- [ ] Monitor Redis server metrics

## Best Practices

### Development
1. **Use appropriate cache strategies** for different widget types
2. **Implement proper error handling** for cache failures
3. **Test cache invalidation logic** thoroughly
4. **Profile widget computation performance**

### Production
1. **Monitor cache hit rates** continuously
2. **Set up proper Redis clustering** for high availability
3. **Use cache warming** to maintain performance
4. **Implement gradual cache invalidation** for large datasets

### Troubleshooting
1. **Check lock status** if widgets seem slow
2. **Monitor Redis memory usage** for potential issues
3. **Review cache hit rates** for optimization opportunities
4. **Analyze response time trends** for performance degradation

## Support

For issues or questions about the caching infrastructure:
1. Check the console command outputs for diagnostics
2. Review Redis logs for connection issues
3. Monitor application logs for cache-related errors
4. Use the status command for health checks

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved