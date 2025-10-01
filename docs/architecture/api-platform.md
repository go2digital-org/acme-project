# API Platform Implementation Guide

## Overview
This guide documents the API Platform 4.x implementation for the ACME Corp CSR platform, covering internationalization support, resource configuration, and integration patterns for headless applications.

## API Route Cleanup
- **Removed all custom routes** from `routes/api.php`
- **Kept only essential endpoints**: SPA authentication and webhook routes
- **API Platform handles all CRUD operations** automatically through resource definitions
- **Eliminated route duplication** and centralized API logic

## Internationalization Features

### Middleware Implementation
- **`ApiLocaleMiddleware`**: Automatic locale detection from `Accept-Language` header
- **Query parameter override**: `?locale=en|nl|fr` support
- **Response headers**: `Content-Language` confirmation
- **Fallback**: Defaults to English for unsupported locales

### Translation Files
Created comprehensive API translation files:
- **English (`en/api.php`)**: Complete message set
- **Dutch (`nl/api.php`)**: Full Dutch translations
- **French (`fr/api.php`)**: Complete French translations

### Supported Languages
- **English (en)** - Default
- **Dutch (nl)** - Netherlands/Belgium
- **French (fr)** - France/Belgium

## Error Handling & Responses

### Standardized Error Format
```json
{
  "success": false,
  "error": {
    "type": "validation_error",
    "message": "De opgegeven gegevens zijn ongeldig",
    "code": "VALIDATION_FAILED",
    "details": {...}
  },
  "meta": {
    "locale": "nl",
    "timestamp": "2024-01-01T12:00:00Z",
    "request_id": "req_123456789"
  }
}
```

### Exception Handler
- **`LocalizedApiExceptionHandler`**: Handles all API exceptions
- **Localized messages**: Based on detected locale
- **Request tracking**: Unique request IDs for debugging
- **Environment awareness**: Detailed errors in development only

## CORS Configuration

### Enhanced CORS Settings
- **Development origins**: `localhost:3000`, `localhost:5173`, etc.
- **Pattern matching**: Supports any localhost port
- **Proper headers**: `Accept-Language`, `Authorization`, `Content-Type`
- **Exposed headers**: `Content-Language`, `X-RateLimit-*`, etc.
- **Credentials support**: Enabled for authenticated requests

## API Platform Resources

### Enhanced Resource Features
All resources now include:
- **Locale middleware**: `api.locale` for language detection
- **Locale filtering**: `?locale=en` query parameter
- **Performance middleware**: Response time monitoring
- **Rate limiting**: Per-endpoint throttling

### Updated Resources
1. **CampaignResource**: Locale-aware with enhanced filtering
2. **DonationResource**: Multilingual support with payment context
3. **OrganizationResource**: Localized organization data
4. **EmployeeResource**: User profile with locale preferences

## Performance Optimizations

### Database Optimization
- **Strategic indexing**: 15+ indexes for common query patterns
- **Full-text search**: MySQL FULLTEXT indexes for search fields
- **Composite indexes**: Multi-field indexes for complex queries
- **Query optimization**: Analyzer table statistics

### Caching Strategy
- **Query caching**: 5-10 minute cache for stable data
- **ETags**: Conditional requests for unchanged content
- **Cache headers**: Appropriate `Cache-Control` directives
- **CDN ready**: `s-maxage` for edge caching

### Large Dataset Handling
- **`OptimizedCollectionProvider`**: Base class for efficient queries
- **Eager loading**: Prevents N+1 query problems
- **Index hints**: Database optimizer hints for large datasets
- **Pagination**: Efficient pagination for 20,000+ records

## Performance Monitoring

### `ApiPerformanceMiddleware`
- **Response time tracking**: Millisecond precision
- **Memory usage monitoring**: Peak and delta memory tracking
- **Slow query logging**: Automatic logging for requests >1s
- **Performance headers**: `X-Response-Time`, `X-Memory-Usage`
- **Development warnings**: Optimization suggestions in dev environment

### Performance Headers
```
X-Request-ID: req_abc123
X-Response-Time: 45.23ms
X-Memory-Usage: 12.5MB
X-Peak-Memory: 15.2MB
Content-Language: nl
Cache-Control: public, max-age=300, s-maxage=600
```

## OpenAPI Documentation

### Enhanced Documentation
- **`OpenApiDocumentationDecorator`**: Automatically enhances API docs
- **Internationalization examples**: Response examples in all languages
- **Language parameters**: `Accept-Language` header documentation
- **Error examples**: Localized error response examples
- **Multi-environment**: Production, staging, development servers

### Example Documentation Features
- **Language negotiation**: How to request specific languages
- **Error handling**: Examples of localized error responses
- **Authentication**: SPA token-based authentication flows
- **Rate limiting**: Understanding rate limit headers

## Authentication & Security

### SPA-Ready Authentication
- **Token-based**: Sanctum personal access tokens
- **Logout handling**: Token revocation on logout
- **Security headers**: Proper CORS and CSP headers
- **Rate limiting**: Separate limits for auth endpoints

### Security Features
- **Input sanitization**: Filter validation for all endpoints
- **SQL injection prevention**: Parameterized queries only
- **XSS protection**: JSON-only responses
- **CSRF protection**: SPA-compatible CSRF handling

## Scalability Features

### Enterprise Readiness
- **20,000+ employee support**: Optimized for large organizations
- **Horizontal scaling**: Stateless API design
- **CDN compatibility**: Proper cache headers
- **Load balancer ready**: Health checks and proper routing

### Monitoring & Debugging
- **Request tracing**: Unique request IDs
- **Performance logging**: Slow query detection
- **Error tracking**: Structured error logging
- **Health checks**: Application health endpoints

## Implementation Files

### Core Files Created/Modified
1. **`routes/api.php`** - Cleaned up, minimal SPA routes only
2. **`app/Http/Middleware/ApiLocaleMiddleware.php`** - Language detection
3. **`app/Http/Middleware/ApiPerformanceMiddleware.php`** - Performance monitoring
4. **`config/cors.php`** - Enhanced CORS configuration
5. **`config/api-platform.php`** - Updated with middleware stack

### Translation Files
- **`lang/en/api.php`** - English API messages
- **`lang/nl/api.php`** - Dutch API messages  
- **`lang/fr/api.php`** - French API messages

### Performance & Documentation
- **`modules/Shared/Infrastructure/ApiPlatform/Provider/OptimizedCollectionProvider.php`** - Base optimization class
- **`modules/Shared/Infrastructure/ApiPlatform/Documentation/OpenApiDocumentationDecorator.php`** - Enhanced API docs
- **`modules/Shared/Infrastructure/ApiPlatform/Exception/LocalizedApiExceptionHandler.php`** - Localized errors
- **`database/indexes.sql`** - Performance indexes

## Results Achieved

### Complete API Platform Migration
- Removed 90+ lines of custom routes
- API Platform handles all CRUD operations automatically
- Centralized resource configuration
- Eliminated code duplication

### Full Internationalization
- 3 languages supported (en, nl, fr)
- Automatic language detection
- Localized error messages
- Response language confirmation

### SPA/Headless Ready
- Token-based authentication
- Proper CORS configuration
- JSON-only responses
- CDN-compatible caching

### Enterprise Performance
- Optimized for 20,000+ records
- Sub-100ms response times (typical)
- Efficient pagination
- Database optimization

### Developer Experience
- Enhanced OpenAPI documentation
- Performance monitoring
- Detailed error responses
- Development debugging tools

## Usage Examples

### Language Negotiation
```bash
# Request Dutch content
curl -H "Accept-Language: nl" https://api.yourdomain.com/v1/campaigns

# Force French via query parameter
curl https://api.yourdomain.com/v1/campaigns?locale=fr

# Complex language preferences
curl -H "Accept-Language: nl,fr;q=0.9,en;q=0.8" https://api.yourdomain.com/v1/campaigns
```

### Authentication Flow
```bash
# Login
curl -X POST https://api.yourdomain.com/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "Accept-Language: nl" \
  -d '{"email":"user@example.com","password":"[password]"}'

# Use token
curl -H "Authorization: Bearer {token}" \
     -H "Accept-Language: nl" \
     https://api.yourdomain.com/v1/campaigns
```

### Performance Monitoring
```bash
# Check response performance
curl -v https://api.yourdomain.com/v1/campaigns | grep "X-Response-Time"
# X-Response-Time: 23.45ms
```

The API Platform provides a production-ready, internationalized API layer supporting multiple frontend implementations including SPAs, mobile apps, and third-party integrations.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved