# API Platform Architecture Design

## Strategic API Design Decision

The integration of API Platform into our hexagonal architecture represents a strategic technical leadership decision that balances enterprise standards with architectural purity. This document details the rationale, implementation strategy, and integration approach.

## Why API Platform?

### Enterprise API Standards

API Platform provides enterprise-grade API functionality out of the box:

```php
// Automatic OpenAPI 3.0 documentation generation
// GET /api/campaigns returns standardized JSON:API format
{
  \"@context\": \"/api/contexts/Campaign\",
  \"@id\": \"/api/campaigns\",
  \"@type\": \"hydra:Collection\",
  \"hydra:member\": [
    {
      \"@id\": \"/api/campaigns/1\",
      \"@type\": \"Campaign\",
      \"id\": 1,
      \"name\": \"Clean Water Initiative\",
      \"goal\": {
        \"amount\": 1000,
        \"currency\": \"EUR\"
      },
      \"status\": \"active\"
    }
  ],
  \"hydra:totalItems\": 1,
  \"hydra:view\": {
    \"@id\": \"/api/campaigns?page=1\",
    \"@type\": \"hydra:PartialCollectionView\"
  }
}
```

### Automatic Documentation Generation

API Platform generates comprehensive documentation:

```yaml
# Generated OpenAPI specification
paths:
  /api/campaigns:
    get:
      tags: [Campaign]
      summary: Retrieves the collection of Campaign resources
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            minimum: 1
        - name: category
          in: query
          schema:
            type: string
        - name: organization
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Campaign collection
          content:
            application/ld+json:
              schema:
                type: object
                properties:
                  'hydra:member':
                    type: array
                    items:
                      $ref: '#/components/schemas/Campaign'
```

## Integration with Hexagonal Architecture

### Custom State Processors

We integrate API Platform with our hexagonal architecture using custom state processors:

```php
// modules/Campaign/Infrastructure/ApiPlatform/State/CampaignCreateProcessor.php
declare(strict_types=1);

namespace Modules\\Campaign\\Infrastructure\\ApiPlatform\\State;

use ApiPlatform\\Metadata\\Operation;
use ApiPlatform\\State\\ProcessorInterface;
use Modules\\Campaign\\Application\\Command\\CreateCampaignCommand;
use Modules\\Campaign\\Infrastructure\\ApiPlatform\\Resource\\CampaignResource;
use Modules\\Shared\\Application\\Command\\CommandBusInterface;

final readonly class CampaignCreateProcessor implements ProcessorInterface
{
    public function __construct(
        private CommandBusInterface $commandBus
    ) {}

    public function process(
        mixed $data,
        Operation $operation,
        array $uriVariables = [],
        array $context = []
    ): CampaignResource {
        assert($data instanceof CampaignResource);
        
        $command = new CreateCampaignCommand(
            name: $data->name,
            description: $data->description,
            goalAmount: $data->goal->amount,
            currency: $data->goal->currency,
            userId: $context['user']->getId(),
            organizationId: $data->organizationId
        );
        
        $campaign = $this->commandBus->dispatch($command);
        
        return CampaignResource::fromDomain($campaign);
    }
}
```

### Resource Mapping Strategy

API Platform resources act as DTOs between HTTP and domain layers:

```php
// modules/Campaign/Infrastructure/ApiPlatform/Resource/CampaignResource.php
declare(strict_types=1);

namespace Modules\\Campaign\\Infrastructure\\ApiPlatform\\Resource;

use ApiPlatform\\Metadata\\ApiResource;
use ApiPlatform\\Metadata\\Get;
use ApiPlatform\\Metadata\\GetCollection;
use ApiPlatform\\Metadata\\Post;
use ApiPlatform\\Metadata\\Put;
use ApiPlatform\\Metadata\\Delete;
use Modules\\Campaign\\Domain\\Model\\Campaign;
use Modules\\Campaign\\Infrastructure\\ApiPlatform\\State\\CampaignCreateProcessor;
use Modules\\Campaign\\Infrastructure\\ApiPlatform\\State\\CampaignProvider;

#[ApiResource(
    shortName: 'Campaign',
    operations: [
        new Get(
            provider: CampaignProvider::class
        ),
        new GetCollection(
            provider: CampaignProvider::class
        ),
        new Post(
            processor: CampaignCreateProcessor::class,
            security: \"is_granted('ROLE_USER')\"
        ),
        new Put(
            processor: CampaignUpdateProcessor::class,
            security: \"is_granted('ROLE_USER') and object.ownerId == user.getId()\"
        ),
        new Delete(
            processor: CampaignDeleteProcessor::class,
            security: \"is_granted('ROLE_USER') and object.ownerId == user.getId()\"
        )
    ],
    normalizationContext: ['groups' => ['campaign:read']],
    denormalizationContext: ['groups' => ['campaign:write']]
)]
class CampaignResource
{
    public function __construct(
        public readonly ?string $id = null,
        public readonly ?string $name = null,
        public readonly ?string $description = null,
        public readonly ?MoneyResource $goal = null,
        public readonly ?MoneyResource $raised = null,
        public readonly ?string $status = null,
        public readonly ?string $userId = null,
        public readonly ?string $organizationId = null,
        public readonly ?\\DateTimeImmutable $createdAt = null
    ) {}
    
    public static function fromDomain(Campaign $campaign): self
    {
        return new self(
            id: $campaign->getId()->toString(),
            name: $campaign->getName(),
            description: $campaign->getDescription(),
            goal: MoneyResource::fromDomain($campaign->getGoal()),
            raised: MoneyResource::fromDomain($campaign->getRaised()),
            status: $campaign->getStatus()->toString(),
            userId: $campaign->getUserId()->toString(),
            organizationId: $campaign->getOrganizationId()->toString(),
            createdAt: $campaign->getCreatedAt()
        );
    }
}
```

### Custom Providers for Queries

Providers delegate to our CQRS query handlers:

```php
// modules/Campaign/Infrastructure/ApiPlatform/State/CampaignProvider.php
declare(strict_types=1);

namespace Modules\\Campaign\\Infrastructure\\ApiPlatform\\State;

use ApiPlatform\\Metadata\\CollectionOperationInterface;
use ApiPlatform\\Metadata\\Operation;
use ApiPlatform\\State\\ProviderInterface;
use Modules\\Campaign\\Application\\Query\\FindCampaignByIdQuery;
use Modules\\Campaign\\Application\\Query\\FindActiveCampaignsQuery;
use Modules\\Campaign\\Infrastructure\\ApiPlatform\\Resource\\CampaignResource;
use Modules\\Shared\\Application\\Query\\QueryBusInterface;

final readonly class CampaignProvider implements ProviderInterface
{
    public function __construct(
        private QueryBusInterface $queryBus
    ) {}

    public function provide(
        Operation $operation,
        array $uriVariables = [],
        array $context = []
    ): CampaignResource|array|null {
        
        if ($operation instanceof CollectionOperationInterface) {
            return $this->provideCollection($context);
        }
        
        return $this->provideItem($uriVariables['id'] ?? null);
    }
    
    private function provideCollection(array $context): array
    {
        $query = new FindActiveCampaignsQuery(
            page: $context['filters']['page'] ?? 1,
            limit: 20,
            category: $context['filters']['category'] ?? null,
            organizationId: $context['filters']['organization'] ?? null
        );
        
        $campaigns = $this->queryBus->dispatch($query);
        
        return array_map(
            fn (Campaign $campaign) => CampaignResource::fromDomain($campaign),
            $campaigns->toArray()
        );
    }
    
    private function provideItem(string $id): ?CampaignResource
    {
        $query = new FindCampaignByIdQuery($id);
        $campaign = $this->queryBus->dispatch($query);
        
        return $campaign ? CampaignResource::fromDomain($campaign) : null;
    }
}
```

## Advanced Filtering and Search

### Custom Filters

```php
// modules/Shared/Infrastructure/ApiPlatform/Filter/MultiSearchFilter.php
declare(strict_types=1);

namespace Modules\\Shared\\Infrastructure\\ApiPlatform\\Filter;

use ApiPlatform\\Doctrine\\Orm\\Filter\\AbstractFilter;
use ApiPlatform\\Doctrine\\Orm\\Util\\QueryNameGeneratorInterface;
use ApiPlatform\\Metadata\\Operation;
use Doctrine\\ORM\\QueryBuilder;

final class MultiSearchFilter extends AbstractFilter
{
    public function getDescription(string $resourceClass): array
    {
        return [
            'search' => [
                'property' => 'search',
                'type' => 'string',
                'required' => false,
                'description' => 'Search across multiple fields'
            ]
        ];
    }
    
    protected function filterProperty(
        string $property,
        mixed $value,
        QueryBuilder $queryBuilder,
        QueryNameGeneratorInterface $queryNameGenerator,
        string $resourceClass,
        Operation $operation = null,
        array $context = []
    ): void {
        if ($property !== 'search' || !is_string($value)) {
            return;
        }
        
        $alias = $queryBuilder->getRootAliases()[0];
        $parameterName = $queryNameGenerator->generateParameterName('search');
        
        $queryBuilder
            ->andWhere(sprintf('(%s.name LIKE :%s OR %s.description LIKE :%s)', 
                $alias, $parameterName, $alias, $parameterName))
            ->setParameter($parameterName, '%' . $value . '%');
    }
}
```

### Usage in Resources

```php
#[ApiResource(
    operations: [
        new GetCollection(
            provider: CampaignProvider::class
        )
    ],
    filters: [
        MultiSearchFilter::class,
        'organization' => OrganizationFilter::class,
        'category' => CategoryFilter::class,
        'status' => StatusFilter::class
    ]
)]
class CampaignResource
{
    // Resource definition...
}
```

## Security Integration

### JWT Authentication

```php
// config/packages/api_platform.yaml
api_platform:
    title: 'ACME Corp CSR API'
    version: '1.0.0'
    description: 'Enterprise CSR Donation Platform API'
    
    swagger:
        api_keys:
            JWT:
                name: Authorization
                type: header
                
    openapi:
        contact:
            name: 'Go2Digital Development Team'
            email: 'info@go2digit.al'
        license:
            name: 'Proprietary'
```

### Resource-Level Security

```php
#[ApiResource(
    operations: [
        new Post(
            processor: CampaignCreateProcessor::class,
            security: \"is_granted('ROLE_USER')\",
            securityMessage: 'Only authenticated users can create campaigns'
        ),
        new Put(
            processor: CampaignUpdateProcessor::class,
            security: \"is_granted('ROLE_EMPLOYEE') and object.userId == user.getId()\",
            securityMessage: 'You can only edit your own campaigns'
        ),
        new Delete(
            processor: CampaignDeleteProcessor::class,
            security: \"is_granted('ROLE_ADMIN') or (is_granted('ROLE_EMPLOYEE') and object.userId == user.getId())\",
            securityMessage: 'Only admins or campaign owners can delete campaigns'
        )
    ]
)]
class CampaignResource
```

## Performance Optimizations

### Pagination Strategy

```php
// Custom paginator for large collections
class OptimizedPaginator implements PaginatorInterface
{
    public function __construct(
        private QueryBusInterface $queryBus,
        private int $itemsPerPage = 20
    ) {}
    
    public function getItems(): iterable
    {
        $query = new FindActiveCampaignsQuery(
            page: $this->getCurrentPage(),
            limit: $this->itemsPerPage
        );
        
        $result = $this->queryBus->dispatch($query);
        
        return array_map(
            fn (Campaign $campaign) => CampaignResource::fromDomain($campaign),
            $result->getItems()
        );
    }
    
    public function count(): int
    {
        // Use optimized count query
        $query = new CountActiveCampaignsQuery();
        return $this->queryBus->dispatch($query);
    }
}
```

### Caching Strategy

```php
// Cache provider results
final readonly class CachedCampaignProvider implements ProviderInterface
{
    public function __construct(
        private CampaignProvider $innerProvider,
        private CacheInterface $cache
    ) {}
    
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        if ($operation instanceof CollectionOperationInterface) {
            $cacheKey = 'campaigns_collection_' . md5(serialize($context['filters'] ?? []));
            
            return $this->cache->get($cacheKey, function () use ($operation, $uriVariables, $context) {
                return $this->innerProvider->provide($operation, $uriVariables, $context);
            }, 300); // 5 minutes cache
        }
        
        $id = $uriVariables['id'] ?? null;
        if (!$id) {
            return null;
        }
        
        $cacheKey = 'campaign_' . $id;
        
        return $this->cache->get($cacheKey, function () use ($operation, $uriVariables, $context) {
            return $this->innerProvider->provide($operation, $uriVariables, $context);
        }, 600); // 10 minutes cache
    }
}
```

## Error Handling Strategy

### Custom Exception Handling

```php
// modules/Shared/Infrastructure/ApiPlatform/EventListener/ApiExceptionListener.php
declare(strict_types=1);

namespace Modules\\Shared\\Infrastructure\\ApiPlatform\\EventListener;

use ApiPlatform\\Validator\\Exception\\ValidationException;
use Modules\\Shared\\Domain\\Exception\\DomainException;
use Symfony\\Component\\HttpFoundation\\JsonResponse;
use Symfony\\Component\\HttpKernel\\Event\\ExceptionEvent;
use Symfony\\Component\\HttpKernel\\Exception\\HttpException;

final readonly class ApiExceptionListener
{
    public function __invoke(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();
        
        if ($exception instanceof DomainException) {
            $response = new JsonResponse([
                'type' => 'https://tools.ietf.org/html/rfc2616#section-10',
                'title' => 'Business Rule Violation',
                'detail' => $exception->getMessage(),
                'status' => 422
            ], 422);
            
            $event->setResponse($response);
            return;
        }
        
        if ($exception instanceof ValidationException) {
            $violations = [];
            foreach ($exception->getConstraintViolationList() as $violation) {
                $violations[] = [
                    'property' => $violation->getPropertyPath(),
                    'message' => $violation->getMessage(),
                    'code' => $violation->getCode()
                ];
            }
            
            $response = new JsonResponse([
                'type' => 'https://tools.ietf.org/html/rfc2616#section-10',
                'title' => 'Validation Failed',
                'detail' => 'The submitted data failed validation',
                'status' => 400,
                'violations' => $violations
            ], 400);
            
            $event->setResponse($response);
            return;
        }
    }
}
```

## API Versioning Strategy

### Version-Specific Resources

```php
// modules/Campaign/Infrastructure/ApiPlatform/Resource/V2/CampaignResource.php
#[ApiResource(
    uriTemplate: '/v2/campaigns',
    shortName: 'Campaign',
    operations: [
        new GetCollection(
            provider: CampaignV2Provider::class
        ),
        new Post(
            processor: CampaignV2CreateProcessor::class
        )
    ]
)]
class CampaignV2Resource
{
    // V2 resource with additional fields or different structure
    public function __construct(
        public readonly ?string $id = null,
        public readonly ?string $name = null,
        public readonly ?string $description = null,
        public readonly ?MoneyResource $goal = null,
        public readonly ?MoneyResource $raised = null,
        public readonly ?string $status = null,
        public readonly ?array $tags = null, // New in V2
        public readonly ?string $visibility = null, // New in V2
        public readonly ?UserResource $owner = null, // Embedded resource in V2
        public readonly ?\\DateTimeImmutable $createdAt = null
    ) {}
}
```

## Integration Testing

### API Platform Test Utilities

```php
// tests/Api/CampaignApiTest.php
use ApiPlatform\\Symfony\\Bundle\\Test\\ApiTestCase;

class CampaignApiTest extends ApiTestCase
{
    public function test_can_create_campaign(): void
    {
        $client = self::createClient();
        $client->request('POST', '/api/campaigns', [
            'json' => [
                'name' => 'Test Campaign',
                'description' => 'Test Description',
                'goal' => [
                    'amount' => 1000,
                    'currency' => 'EUR'
                ],
                'organizationId' => 'org-123'
            ],
            'headers' => [
                'Authorization' => 'Bearer ' . $this->getAuthToken()
            ]
        ]);
        
        $this->assertResponseStatusCodeSame(201);
        $this->assertJsonContains([
            'name' => 'Test Campaign',
            'status' => 'draft'
        ]);
    }
    
    public function test_can_filter_campaigns_by_category(): void
    {
        $client = self::createClient();
        
        $client->request('GET', '/api/campaigns?category=environment');
        
        $this->assertResponseIsSuccessful();
        $this->assertJsonContains([
            'hydra:totalItems' => 5
        ]);
    }
}
```

## Documentation Generation

### Custom Documentation Provider

```php
// modules/Shared/Infrastructure/ApiPlatform/OpenApi/DocumentationProvider.php
final readonly class DocumentationProvider implements OpenApiFactoryInterface
{
    public function __construct(
        private OpenApiFactoryInterface $decorated
    ) {}
    
    public function __invoke(array $context = []): OpenApi
    {
        $openApi = ($this->decorated)($context);
        
        // Add custom documentation
        $openApi = $openApi->withInfo($openApi->getInfo()->withDescription('
            # ACME Corp CSR Donation Platform API
            
            This API enables users to create and manage fundraising campaigns
            and make donations to causes they believe in.
            
            ## Authentication
            
            All endpoints require JWT authentication via the Authorization header:
            ```
            Authorization: Bearer YOUR_JWT_TOKEN
            ```
            
            ## Rate Limiting
            
            API requests are limited to 1000 requests per hour per authenticated user.
        '));
        
        // Add security schemes
        $openApi = $openApi->withComponents(
            $openApi->getComponents()->withSecuritySchemes([
                'JWT' => new SecurityScheme(
                    type: 'http',
                    scheme: 'bearer',
                    bearerFormat: 'JWT'
                )
            ])
        );
        
        return $openApi;
    }
}
```

## Monitoring and Analytics

### API Usage Metrics

```php
// modules/Shared/Infrastructure/ApiPlatform/EventListener/ApiMetricsListener.php
final readonly class ApiMetricsListener
{
    public function __construct(
        private MetricsCollectorInterface $metricsCollector
    ) {}
    
    public function onKernelResponse(ResponseEvent $event): void
    {
        $request = $event->getRequest();
        $response = $event->getResponse();
        
        if (!str_starts_with($request->getPathInfo(), '/api/')) {
            return;
        }
        
        $this->metricsCollector->increment('api.requests.total', [
            'method' => $request->getMethod(),
            'endpoint' => $this->normalizeEndpoint($request->getPathInfo()),
            'status_code' => $response->getStatusCode()
        ]);
        
        $this->metricsCollector->histogram('api.response_time', 
            microtime(true) - $request->server->get('REQUEST_TIME_FLOAT'),
            ['endpoint' => $this->normalizeEndpoint($request->getPathInfo())]
        );
    }
}
```

## Strategic Benefits

### For Development Teams

1. **Reduced Boilerplate**: API Platform eliminates repetitive CRUD code
2. **Automatic Documentation**: OpenAPI docs stay in sync with implementation
3. **Consistent Standards**: Enforces REST and JSON:API standards
4. **Rich Filtering**: Built-in filtering and pagination capabilities

### For Business Stakeholders

1. **Faster Integration**: Third-party developers can integrate quickly with auto-generated docs
2. **API Consistency**: Standardized API structure across all endpoints
3. **Future-Proof**: Built on web standards and best practices
4. **Enterprise Ready**: Production-grade features like rate limiting and security

### For Enterprise Architecture

1. **Standards Compliance**: Follows JSON:API and OpenAPI specifications
2. **Security Integration**: Built-in authentication and authorization
3. **Performance Optimization**: Efficient serialization and caching
4. **Monitoring Capabilities**: Built-in metrics and logging

The API Platform integration provides a production-ready API layer with automatic OpenAPI documentation, standardized response formats, and enterprise-grade features while maintaining hexagonal architecture boundaries.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved