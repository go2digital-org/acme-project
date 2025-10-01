# ADR-002: API Platform over Traditional Laravel Routes

## Status

Accepted

## Context

The ACME Corp CSR Platform requires a robust API layer to support:
- Mobile applications and SPAs
- Third-party integrations
- Admin dashboard functionality
- Webhook endpoints for external services

We needed to choose between different approaches for API development:

1. **Traditional Laravel Routes with Controllers**
   - Manual route definition in `routes/api.php`
   - Custom controller methods for each endpoint
   - Manual validation and serialization

2. **Laravel API Resources with Route Model Binding**
   - Structured response formatting
   - Still requires manual route definition
   - Limited automatic documentation

3. **API Platform**
   - Convention-over-configuration approach
   - Automatic OpenAPI documentation
   - Built-in filtering, pagination, and validation
   - Hypermedia support (JSON-LD, HAL)

## Decision

We will use **API Platform** as our primary API framework, integrated with our hexagonal architecture.

### Key Features Utilized:
- **Automatic API Generation**: Based on domain models and configuration
- **OpenAPI Documentation**: Auto-generated and always up-to-date
- **Standard Compliance**: JSON-LD, Hydra, and HAL support
- **Advanced Filtering**: Built-in search, filter, and pagination
- **Validation Integration**: Works with Laravel's validation system
- **Security**: Built-in authentication and authorization patterns

### Integration Approach:
```php
// Domain Model with API Platform annotations
#[ApiResource(
    operations: [
        new GetCollection(),
        new Get(),
        new Post(security: "is_granted('ROLE_ADMIN')"),
        new Put(security: "is_granted('ROLE_ADMIN')"),
        new Delete(security: "is_granted('ROLE_ADMIN')")
    ],
    normalizationContext: ['groups' => ['campaign:read']],
    denormalizationContext: ['groups' => ['campaign:write']]
)]
class Campaign
{
    // Domain model implementation
}
```

## Consequences

### Positive:
- **Rapid Development**: Automatic CRUD operations generation
- **Consistency**: Standardized API responses across all endpoints
- **Documentation**: Always up-to-date OpenAPI specifications
- **Standards Compliance**: REST and hypermedia best practices
- **Client Generation**: Automatic client library generation
- **Testing**: Built-in API testing capabilities
- **Performance**: Optimized queries and caching out of the box

### Negative:
- **Learning Curve**: Team needs to learn API Platform concepts
- **Less Control**: Some customizations require deeper framework knowledge
- **Framework Dependency**: Adds another layer of abstraction
- **Complex Queries**: Advanced business logic still requires custom implementations

### Risk Mitigation:
- **Custom Operations**: Use custom API Platform operations for complex business logic
- **Fallback Routes**: Traditional Laravel routes for specific edge cases
- **Gradual Migration**: Implement API Platform incrementally
- **Documentation**: Comprehensive guides for common patterns

## Implementation Strategy

### Phase 1: Core Resources
- Campaign API with full CRUD operations
- User management endpoints
- Basic filtering and pagination

### Phase 2: Advanced Features
- Custom operations for business logic
- Advanced filtering and search
- File upload endpoints

### Phase 3: Integration
- Webhook endpoints
- Third-party API integrations
- Mobile app optimizations

### Configuration Example:
```yaml
# config/packages/api_platform.yaml
api_platform:
    mapping:
        paths: ['%kernel.project_dir%/src/Modules/*/Domain/Model']
    patch_formats:
        json: ['application/merge-patch+json']
    swagger:
        versions: [3]
    defaults:
        pagination_enabled: true
        pagination_page_size: 20
        pagination_maximum_items_per_page: 100
```

## Custom Operations

For complex business logic that doesn't fit CRUD patterns:

```php
#[ApiResource(
    operations: [
        new Post(
            uriTemplate: '/campaigns/{id}/donate',
            controller: DonateToCampaignController::class,
            security: "is_granted('ROLE_USER')"
        )
    ]
)]
class Campaign
{
    // Custom donation operation
}
```

## Testing Approach

```php
// API Platform test example
public function testCampaignCreation(): void
{
    $response = static::createClient()->request('POST', '/api/campaigns', [
        'json' => [
            'title' => 'Test Campaign',
            'description' => 'Test Description',
            'goalAmount' => 10000
        ],
        'headers' => ['Authorization' => 'Bearer ' . $this->getToken()]
    ]);

    $this->assertResponseStatusCodeSame(201);
    $this->assertJsonContains(['title' => 'Test Campaign']);
}
```

## Related Decisions

- [ADR-001: Hexagonal Architecture](001-hexagonal-architecture.md) - Overall architecture pattern
- [ADR-003: CQRS Pattern](003-cqrs-pattern.md) - Application layer organization

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved