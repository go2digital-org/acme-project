# API Documentation

## Overview

The ACME Corp CSR Platform implements a pure API-first architecture using API Platform 4.x with zero traditional Laravel routes. This design ensures enterprise-grade API standards with automatic OpenAPI 3.0 documentation, JSON:API compliance, and complete technology independence.

## API Architecture

### Pure API Platform Implementation

The platform eliminates traditional Laravel routes in favor of API Platform resources:

- **State Processors**: Handle write operations (CREATE, UPDATE, DELETE)
- **State Providers**: Handle read operations (GET, LIST)
- **Resource Definitions**: Define API structure and behavior
- **Automatic Documentation**: OpenAPI 3.0 spec generation
- **JSON:API Compliance**: Standardized response format

### Base URL and Access

- **Development**: `http://localhost:8000/api`
- **Docker**: `http://app.acme-corp-optimy.orb.local/api`
- **Interactive Documentation**: `http://localhost:8000/api` (Swagger UI)
- **OpenAPI Specification**: `http://localhost:8000/api/docs.json`

## Authentication

### JWT Token Authentication

All API endpoints require Bearer token authentication:

```http
Authorization: Bearer {jwt_token}
Content-Type: application/json
Accept: application/json
```

### Authentication Endpoints

#### User Login
```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
```

**Response (200 OK):**
```json
{
  "data": {
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "first_name": "John",
      "last_name": "Doe",
      "role": "user"
    }
  }
}
```

#### Token Refresh
```http
POST /api/auth/refresh
Authorization: Bearer {current_token}
```

#### User Logout
```http
POST /api/auth/logout
Authorization: Bearer {token}
```

## API Resources

### Campaign Management

#### List Campaigns
```http
GET /api/campaigns
Authorization: Bearer {token}
```

**Query Parameters:**
- `page` (integer): Page number for pagination
- `itemsPerPage` (integer): Items per page (default: 30)
- `status` (string): Filter by status (active, draft, completed, cancelled)
- `category` (string): Filter by category
- `organization_id` (string): Filter by organization
- `search` (string): Search in title and description

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Clean Water Initiative",
      "description": "Providing clean water access to rural communities",
      "target_amount": 50000.00,
      "currency": "EUR",
      "raised_amount": 12500.00,
      "status": "active",
      "category": "environment",
      "start_date": "2025-01-01T00:00:00Z",
      "end_date": "2025-12-31T23:59:59Z",
      "organization": {
        "id": "uuid",
        "name": "ACME Corp",
        "slug": "acme-corp"
      },
      "created_by": {
        "id": "uuid",
        "first_name": "John",
        "last_name": "Doe"
      },
      "created_at": "2025-01-01T10:00:00Z",
      "updated_at": "2025-01-15T14:30:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 5,
    "per_page": 30,
    "to": 30,
    "total": 150
  },
  "links": {
    "first": "/api/campaigns?page=1",
    "last": "/api/campaigns?page=5",
    "prev": null,
    "next": "/api/campaigns?page=2"
  }
}
```

#### Get Single Campaign
```http
GET /api/campaigns/{id}
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "data": {
    "id": "uuid",
    "title": "Clean Water Initiative",
    "description": "Detailed description of the campaign...",
    "target_amount": 50000.00,
    "currency": "EUR",
    "raised_amount": 12500.00,
    "status": "active",
    "category": "environment",
    "start_date": "2025-01-01T00:00:00Z",
    "end_date": "2025-12-31T23:59:59Z",
    "progress_percentage": 25.0,
    "donations_count": 42,
    "organization": {
      "id": "uuid",
      "name": "ACME Corp",
      "slug": "acme-corp"
    },
    "created_by": {
      "id": "uuid",
      "first_name": "John",
      "last_name": "Doe",
      "email": "john.doe@acme.com"
    },
    "recent_donations": [
      {
        "id": "uuid",
        "amount": 100.00,
        "currency": "EUR",
        "donor_name": "Anonymous",
        "message": "Great cause!",
        "created_at": "2025-01-15T14:30:00Z"
      }
    ],
    "created_at": "2025-01-01T10:00:00Z",
    "updated_at": "2025-01-15T14:30:00Z"
  }
}
```

#### Create Campaign
```http
POST /api/campaigns
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "New Environmental Campaign",
  "description": "Detailed description of the campaign objectives and goals",
  "target_amount": 25000.00,
  "currency": "EUR",
  "category": "environment",
  "start_date": "2025-02-01",
  "end_date": "2025-11-30",
  "organization_id": "uuid"
}
```

**Response (201 Created):**
```json
{
  "data": {
    "id": "new-uuid",
    "title": "New Environmental Campaign",
    "status": "draft",
    "target_amount": 25000.00,
    "currency": "EUR",
    "raised_amount": 0.00,
    "created_at": "2025-01-18T10:00:00Z"
  }
}
```

#### Update Campaign
```http
PUT /api/campaigns/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Updated Campaign Title",
  "description": "Updated description",
  "target_amount": 30000.00
}
```

#### Delete Campaign
```http
DELETE /api/campaigns/{id}
Authorization: Bearer {token}
```

**Response (204 No Content)**

### Donation Management

#### Create Donation
```http
POST /api/donations
Authorization: Bearer {token}
Content-Type: application/json

{
  "campaign_id": "uuid",
  "amount": 100.00,
  "currency": "EUR",
  "payment_method": "credit_card",
  "anonymous": false,
  "message": "Keep up the great work!",
  "donor_details": {
    "first_name": "Jane",
    "last_name": "Smith",
    "email": "jane.smith@email.com"
  }
}
```

**Response (201 Created):**
```json
{
  "data": {
    "id": "uuid",
    "amount": 100.00,
    "currency": "EUR",
    "status": "pending",
    "payment_intent_id": "pi_1234567890",
    "campaign": {
      "id": "uuid",
      "title": "Clean Water Initiative"
    },
    "created_at": "2025-01-18T10:00:00Z"
  }
}
```

#### List Donations
```http
GET /api/donations
Authorization: Bearer {token}
```

**Query Parameters:**
- `campaign_id` (string): Filter by campaign
- `status` (string): Filter by status (pending, completed, failed, refunded)
- `date_from` (date): Filter donations from date
- `date_to` (date): Filter donations to date

### Organization Management

#### List Organizations
```http
GET /api/organizations
Authorization: Bearer {token}
```

#### Get Organization Details
```http
GET /api/organizations/{id}
Authorization: Bearer {token}
```

#### Update Organization
```http
PUT /api/organizations/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Organization Name",
  "description": "Updated description",
  "website": "https://example.com",
  "contact_email": "contact@example.com"
}
```

### User Management

#### Get Current User Profile
```http
GET /api/user/profile
Authorization: Bearer {token}
```

#### Update User Profile
```http
PUT /api/user/profile
Authorization: Bearer {token}
Content-Type: application/json

{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john.doe@example.com",
  "phone": "+1234567890",
  "bio": "Updated bio information"
}
```

#### Change Password
```http
PUT /api/user/password
Authorization: Bearer {token}
Content-Type: application/json

{
  "current_password": "CurrentPassword123!",
  "password": "NewPassword123!",
  "password_confirmation": "NewPassword123!"
}
```

### Search API

#### Global Search
```http
GET /api/search
Authorization: Bearer {token}
```

**Query Parameters:**
- `q` (string, required): Search query
- `type` (string): Filter by type (campaigns, organizations, users)
- `limit` (integer): Results limit (default: 20)

**Response (200 OK):**
```json
{
  "data": {
    "campaigns": [
      {
        "id": "uuid",
        "title": "Clean Water Initiative",
        "description": "Providing clean water...",
        "target_amount": 50000.00,
        "raised_amount": 12500.00,
        "status": "active"
      }
    ],
    "organizations": [
      {
        "id": "uuid",
        "name": "ACME Corp",
        "description": "Corporate social responsibility..."
      }
    ],
    "total_results": 25,
    "search_time": 0.045
  }
}
```

### Currency API

#### Get Supported Currencies
```http
GET /api/currencies
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "code": "EUR",
      "name": "Euro",
      "symbol": "€",
      "decimal_places": 2,
      "is_default": true
    },
    {
      "code": "USD",
      "name": "US Dollar",
      "symbol": "$",
      "decimal_places": 2,
      "is_default": false
    }
  ]
}
```

#### Currency Conversion
```http
GET /api/currencies/convert
Authorization: Bearer {token}
```

**Query Parameters:**
- `from` (string, required): Source currency code
- `to` (string, required): Target currency code
- `amount` (number, required): Amount to convert

## Response Formats

### Standard Response Structure

All API responses follow a consistent JSON:API structure:

```json
{
  "data": {
    // Resource data or array of resources
  },
  "meta": {
    // Metadata (pagination, counts, etc.)
  },
  "links": {
    // Pagination links
  },
  "included": [
    // Related resources (when using include parameter)
  ]
}
```

### Error Response Structure

```json
{
  "errors": [
    {
      "status": "422",
      "title": "Validation Error",
      "detail": "The title field is required.",
      "source": {
        "pointer": "/data/attributes/title"
      }
    }
  ]
}
```

## HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | OK - Request successful |
| 201 | Created - Resource created successfully |
| 204 | No Content - Request successful, no content returned |
| 400 | Bad Request - Invalid request data |
| 401 | Unauthorized - Authentication required |
| 403 | Forbidden - Insufficient permissions |
| 404 | Not Found - Resource not found |
| 422 | Unprocessable Entity - Validation errors |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error - Server error |

## Rate Limiting

API requests are rate-limited to ensure fair usage:

- **Authenticated users**: 1000 requests per hour
- **Guest users**: 100 requests per hour
- **Admin users**: 2000 requests per hour

Rate limit headers are included in responses:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1642598400
```

## Pagination

### Query Parameters

- `page` (integer): Page number (default: 1)
- `itemsPerPage` (integer): Items per page (default: 30, max: 100)

### Response Meta

```json
{
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 5,
    "per_page": 30,
    "to": 30,
    "total": 150
  },
  "links": {
    "first": "/api/campaigns?page=1",
    "last": "/api/campaigns?page=5",
    "prev": null,
    "next": "/api/campaigns?page=2"
  }
}
```

## Filtering and Sorting

### Filter Parameters

Most collection endpoints support filtering:

```http
GET /api/campaigns?status=active&category=environment&organization_id=uuid
```

### Sorting

Use the `sort` parameter with field names:

```http
GET /api/campaigns?sort=created_at,-target_amount
```

- Prefix with `-` for descending order
- Multiple sort fields supported

### Including Related Resources

Use the `include` parameter to include related resources:

```http
GET /api/campaigns?include=organization,created_by
```

## API Platform Features

### Automatic Documentation

- **Interactive Swagger UI**: Available at `/api`
- **OpenAPI 3.0 Specification**: Available at `/api/docs.json`
- **JSON-LD Context**: Available at `/api/contexts/{resource}`

### Content Negotiation

Supported formats:
- `application/json` (default)
- `application/ld+json` (JSON-LD)
- `text/html` (API documentation)

### Validation

Request validation is automatic based on resource definitions. Validation errors return detailed information:

```json
{
  "errors": [
    {
      "status": "422",
      "title": "Validation Error",
      "detail": "The email field must be a valid email address.",
      "source": {
        "pointer": "/data/attributes/email"
      }
    }
  ]
}
```

## Security Features

### JWT Authentication

- Token-based authentication using JSON Web Tokens
- Automatic token refresh mechanism
- Secure token storage recommendations

### Rate Limiting

- Per-user rate limiting
- Global rate limiting
- Custom rate limits for different user types

### Input Validation

- Automatic request validation
- SQL injection prevention
- XSS protection
- CSRF protection for web routes

### CORS Support

Configured CORS headers for cross-origin requests:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Requested-With
```

## Error Handling

### Common Error Scenarios

#### Authentication Errors
```json
{
  "errors": [
    {
      "status": "401",
      "title": "Unauthorized",
      "detail": "Invalid or expired token"
    }
  ]
}
```

#### Validation Errors
```json
{
  "errors": [
    {
      "status": "422",
      "title": "Validation Error",
      "detail": "The target_amount field must be greater than 0.",
      "source": {
        "pointer": "/data/attributes/target_amount"
      }
    }
  ]
}
```

#### Not Found Errors
```json
{
  "errors": [
    {
      "status": "404",
      "title": "Not Found",
      "detail": "Campaign not found"
    }
  ]
}
```

## API Testing

### Using curl

```bash
# Get campaigns
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Accept: application/json" \
     http://localhost:8000/api/campaigns

# Create campaign
curl -X POST \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d '{"title":"Test Campaign","target_amount":1000,"currency":"EUR"}' \
     http://localhost:8000/api/campaigns
```

### Using Postman

Import the OpenAPI specification from `/api/docs.json` to automatically generate a Postman collection.

### Using PHPUnit/Pest

```php
test('creates campaign via API', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->postJson('/api/campaigns', [
            'title' => 'Test Campaign',
            'target_amount' => 1000,
            'currency' => 'EUR'
        ]);

    $response->assertStatus(201);
    $response->assertJsonStructure([
        'data' => ['id', 'title', 'target_amount']
    ]);
});
```

## Best Practices

### API Usage

1. **Use appropriate HTTP methods**: GET for reading, POST for creating, PUT for updating, DELETE for removing
2. **Include proper headers**: Always include Accept and Content-Type headers
3. **Handle errors gracefully**: Check response status and handle error responses
4. **Respect rate limits**: Implement backoff strategies for rate-limited requests
5. **Use pagination**: Don't load all results at once for large datasets

### Performance

1. **Use filtering**: Apply filters to reduce response size
2. **Use includes wisely**: Only include related resources when needed
3. **Cache responses**: Implement client-side caching for static data
4. **Batch requests**: Group related operations when possible

### Security

1. **Secure token storage**: Store JWT tokens securely
2. **Validate responses**: Always validate API responses
3. **Use HTTPS**: Never send tokens over unencrypted connections
4. **Implement logout**: Properly invalidate tokens on logout

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved