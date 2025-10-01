# Go2digit.al ACME Corp CSR Platform - API Documentation

## Overview

The Go2digit.al ACME Corp CSR Platform provides a comprehensive RESTful API built with API Platform, offering automatic OpenAPI 3.0 specification generation and interactive documentation.

## Accessing API Documentation

### Interactive Documentation
- **OpenAPI UI**: [http://localhost:8000/api](http://localhost:8000/api)
- **Swagger UI**: Auto-generated from API Platform resources
- **Format**: OpenAPI 3.0 / JSON-LD / HAL

### Authentication
All API endpoints require JWT authentication except public endpoints.

```bash
# Obtain JWT token
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password"
}

# Use token in requests
Authorization: Bearer {token}
```

## API Resources

### Campaign Management

#### Endpoints
- `GET /api/campaigns` - List all campaigns
- `GET /api/campaigns/{id}` - Get campaign details
- `POST /api/campaigns` - Create new campaign
- `PUT /api/campaigns/{id}` - Update campaign
- `DELETE /api/campaigns/{id}` - Delete campaign
- `GET /api/campaigns/{id}/donations` - Get campaign donations

#### Resource Schema
```json
{
  "@context": "/api/contexts/Campaign",
  "@id": "/api/campaigns/1",
  "@type": "Campaign",
  "id": 1,
  "title": "string",
  "description": "string",
  "goal_amount": 10000,
  "current_amount": 5000,
  "status": "active",
  "start_date": "2024-01-01T00:00:00+00:00",
  "end_date": "2024-12-31T23:59:59+00:00",
  "category_id": 1,
  "organization_id": 1,
  "donations": "/api/campaigns/1/donations"
}
```

### Donation Processing

#### Endpoints
- `GET /api/donations` - List donations
- `GET /api/donations/{id}` - Get donation details
- `POST /api/donations` - Create donation
- `POST /api/donations/{id}/confirm` - Confirm payment
- `POST /api/donations/{id}/refund` - Process refund

#### Resource Schema
```json
{
  "@context": "/api/contexts/Donation",
  "@id": "/api/donations/1",
  "@type": "Donation",
  "id": 1,
  "amount": 100.00,
  "currency": "USD",
  "status": "completed",
  "paymentMethod": "stripe",
  "transactionId": "pi_1234567890",
  "campaign": "/api/campaigns/1",
  "donor": "/api/users/1",
  "createdAt": "2024-01-15T10:30:00+00:00",
  "anonymous": false,
  "message": "Keep up the great work!"
}
```

### User Management

#### Endpoints
- `GET /api/users` - List users (admin only)
- `GET /api/users/{id}` - Get user profile
- `POST /api/users` - Register user
- `PUT /api/users/{id}` - Update profile
- `DELETE /api/users/{id}` - Deactivate user
- `GET /api/users/me` - Current user profile

#### Resource Schema
```json
{
  "@context": "/api/contexts/User",
  "@id": "/api/users/1",
  "@type": "User",
  "id": 1,
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "role": "employee",
  "department": "Engineering",
  "organization": "/api/organizations/1",
  "totalDonations": 2500.00,
  "campaignsSupported": 15,
  "joinedAt": "2023-01-01T00:00:00+00:00",
  "verified": true
}
```

### Category Management

#### Endpoints
- `GET /api/categories` - List categories
- `GET /api/categories/{id}` - Get category details
- `POST /api/categories` - Create category (admin)
- `PUT /api/categories/{id}` - Update category (admin)
- `DELETE /api/categories/{id}` - Delete category (admin)

### Organization Management

#### Endpoints
- `GET /api/organizations` - List organizations
- `GET /api/organizations/{id}` - Get organization details
- `POST /api/organizations` - Create organization
- `PUT /api/organizations/{id}` - Update organization
- `GET /api/organizations/{id}/campaigns` - Organization campaigns
- `GET /api/organizations/{id}/employees` - Organization employees

### Analytics & Reporting

#### Endpoints
- `GET /api/analytics/dashboard` - Dashboard metrics
- `GET /api/analytics/campaigns/{id}` - Campaign analytics
- `GET /api/analytics/donations` - Donation trends
- `GET /api/analytics/engagement` - Employee engagement
- `GET /api/reports/generate` - Generate custom report

## API Features

### Pagination
All collection endpoints support pagination:

```bash
GET /api/campaigns?page=1&itemsPerPage=30
```

Response includes pagination metadata:
```json
{
  "@context": "/api/contexts/Campaign",
  "@id": "/api/campaigns",
  "@type": "hydra:Collection",
  "hydra:member": [...],
  "hydra:totalItems": 150,
  "hydra:view": {
    "@id": "/api/campaigns?page=1",
    "@type": "hydra:PartialCollectionView",
    "hydra:first": "/api/campaigns?page=1",
    "hydra:last": "/api/campaigns?page=5",
    "hydra:next": "/api/campaigns?page=2"
  }
}
```

### Filtering
Support for advanced filtering:

```bash
# Filter by status
GET /api/campaigns?status=active

# Filter by date range
GET /api/campaigns?startDate[after]=2024-01-01&endDate[before]=2024-12-31

# Filter by category
GET /api/campaigns?category=/api/categories/1

# Combined filters
GET /api/campaigns?status=active&category=/api/categories/1&order[createdAt]=desc
```

### Sorting
Order results by any field:

```bash
GET /api/campaigns?order[createdAt]=desc
GET /api/campaigns?order[goal_amount]=asc
```

### Field Selection
Optimize response payload:

```bash
GET /api/campaigns?fields[]=id&fields[]=title&fields[]=goal_amount
```

### Search
Full-text search powered by Meilisearch:

```bash
GET /api/campaigns?search=environment
```

## Error Handling

### Error Response Format
```json
{
  "@context": "/api/contexts/Error",
  "@type": "hydra:Error",
  "hydra:title": "An error occurred",
  "hydra:description": "Detailed error message",
  "trace": [...] // Only in development
}
```

### HTTP Status Codes
- `200 OK` - Successful GET/PUT
- `201 Created` - Successful POST
- `204 No Content` - Successful DELETE
- `400 Bad Request` - Invalid request data
- `401 Unauthorized` - Missing/invalid authentication
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource not found
- `422 Unprocessable Entity` - Validation errors
- `429 Too Many Requests` - Rate limit exceeded
- `500 Internal Server Error` - Server error

## Rate Limiting

Default rate limits:
- **Anonymous**: 30 requests/minute
- **Authenticated**: 60 requests/minute
- **Admin**: 120 requests/minute

Rate limit headers:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1640995200
```

## Webhooks

### Available Events
- `campaign.created`
- `campaign.updated`
- `campaign.completed`
- `donation.created`
- `donation.confirmed`
- `donation.refunded`
- `user.registered`
- `user.verified`

### Webhook Payload
```json
{
  "event": "donation.created",
  "timestamp": "2024-01-15T10:30:00+00:00",
  "data": {
    "donation": {...},
    "campaign": {...},
    "user": {...}
  }
}
```

## API Versioning

The API supports versioning via headers:

```bash
Accept: application/vnd.api+json;version=1
```

Current versions:
- `v1` - Current stable version
- `v2` - Beta (breaking changes)

## Testing API

### Development Environment
```bash
# Import Postman collection
curl -o acme-api.postman.json http://localhost:8000/api/docs.json

# Run tests
./vendor/bin/pest tests/Feature/Api/
```

## Security

### Authentication Methods
- **JWT Bearer Token** (recommended)
- **OAuth 2.0** (coming soon)
- **API Key** (deprecated)

### CORS Configuration
```javascript
// Allowed origins
[
  "http://localhost:3000",
  "https://app.yourdomain.com",
  "https://admin.yourdomain.com"
]
```

### Data Encryption
- All API traffic must use HTTPS
- Sensitive data encrypted at rest
- PCI DSS compliant payment processing

## Performance

### Response Times (p95)
- Simple GET: < 50ms
- Complex queries: < 200ms
- Write operations: < 300ms
- Search queries: < 100ms

### Caching
- **CDN**: Static resources
- **Redis**: API responses (60s TTL)
- **ETags**: Client-side caching

## Support

### Documentation
- Interactive API: `/api`
- OpenAPI Spec: `/api/docs.json`
- This guide: `/docs/api/`

### Contact
- Technical Support: info@go2digit.al
- Bug Reports: Contact info@go2digit.al

---

---

**Developed and Maintained by Go2digit.al**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved