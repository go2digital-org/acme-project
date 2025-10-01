# Dashboard Cache Optimization

## Overview

This feature implements async cache warming for user dashboards to handle 2M+ records efficiently. The dashboard loads immediately with loading states while heavy calculations run in the background.

## Architecture

### Cache Keys (User-Specific)
- `user:{userId}:statistics` - User donation statistics
- `user:{userId}:activity_feed` - Recent activity feed
- `user:{userId}:impact_metrics` - Impact calculations
- `user:{userId}:ranking` - Organization ranking
- `user:{userId}:leaderboard` - Top donors leaderboard

### TTL
- User cache: 30 minutes
- Organization cache: 30 minutes

## How It Works

1. **User visits dashboard** Controller checks cache status
2. **Cache miss** Show loading state, dispatch async job
3. **Background job** Calculates heavy queries, stores in cache
4. **Frontend polling** Checks status every 2 seconds
5. **Cache ready** Frontend fetches data via AJAX
6. **Progressive loading** Each widget appears as data becomes available

## API Endpoints

### Check Cache Status
```
GET /dashboard/cache-status
```
Returns:
```json
{
  "status": "hit|miss|warming",
  "ready": true|false,
  "progress": { ... }
}
```

### Get Dashboard Data
```
GET /dashboard/data
```
Returns cached data or 202 if cache not ready

### Warm Cache (API)
```
POST /api/dashboard/cache/warm
```

### Invalidate Cache
```
DELETE /api/dashboard/cache/invalidate
```

## Frontend Integration

The dashboard uses Alpine.js for reactive loading states:
- Shows skeleton loaders during cache warming
- Polls for cache status
- Progressively loads widgets as data becomes available
- No page blocking or timeouts

## Performance Benefits

- **Initial load**: < 100ms (immediate response)
- **Cache warming**: 5-30s in background
- **Cached requests**: < 50ms
- **Memory efficient**: Batch processing
- **Scalable**: Handles 2M+ records

## Cache Invalidation

Cache is automatically invalidated on:
- New donation
- Campaign creation
- Profile updates

## Queue Configuration

Jobs run on `cache-warming` queue:
- Timeout: 10 minutes
- Retries: 3
- Priority: low

## Usage

No special configuration needed. The system automatically:
1. Detects cold cache on dashboard access
2. Shows loading state
3. Warms cache in background
4. Updates UI when ready

## Monitoring

Check cache warming progress:
```bash
php artisan queue:work cache-warming
```

View cache statistics:
```bash
php artisan cache:stats user
```

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved