# ADR-003: CQRS Pattern for Complex Domain Logic

## Status

Accepted

## Context

The ACME Corp CSR Platform handles complex business operations that involve:
- Multi-step campaign creation and approval workflows
- Complex donation processing with multiple payment methods
- Real-time reporting and analytics queries
- Event sourcing for audit trails and compliance

Traditional CRUD operations become insufficient when dealing with:
- **Complex Business Logic**: Campaign approval workflows, donation matching
- **Performance Requirements**: Read-heavy operations (reports) vs write-heavy operations (donations)
- **Different Data Models**: Write operations need normalized data, read operations need denormalized views
- **Scalability**: Read and write operations have different scaling requirements

We evaluated several patterns:
1. **Traditional CRUD**: Simple but doesn't handle complexity well
2. **CQRS (Command Query Responsibility Segregation)**: Separates reads and writes
3. **Event Sourcing**: Complete audit trail but complex implementation
4. **Hybrid Approach**: CQRS for complex operations, CRUD for simple ones

## Decision

We will implement **CQRS pattern** for complex domain operations while maintaining traditional CRUD for simple entities.

### CQRS Implementation:
- **Commands**: Represent business intentions (CreateCampaign, ProcessDonation)
- **Queries**: Retrieve data for specific use cases (CampaignList, DonationReport)
- **Handlers**: Process commands and queries independently
- **Separation**: Different models for reads and writes when beneficial

### Structure:
```
Application/
├── Command/
│   ├── CreateCampaignCommand.php
│   ├── ProcessDonationCommand.php
│   └── ApproveCampaignCommand.php
├── Query/
│   ├── GetCampaignListQuery.php
│   ├── GetDonationReportQuery.php
│   └── GetCampaignDetailsQuery.php
└── Handler/
    ├── Command/
    │   ├── CreateCampaignHandler.php
    │   └── ProcessDonationHandler.php
    └── Query/
        ├── GetCampaignListHandler.php
        └── GetDonationReportHandler.php
```

## Consequences

### Positive:
- **Scalability**: Read and write operations can be optimized independently
- **Flexibility**: Different data models for reads and writes
- **Performance**: Queries can use optimized read models (materialized views)
- **Maintainability**: Clear separation of business operations
- **Testing**: Commands and queries can be tested in isolation
- **Audit Trail**: Commands naturally provide operation history
- **Concurrent Access**: Reduced lock contention between reads and writes

### Negative:
- **Complexity**: More code and abstractions to maintain
- **Eventual Consistency**: Read models might be slightly behind write models
- **Learning Curve**: Team needs to understand CQRS principles
- **Debugging**: More components to trace through

### Risk Mitigation:
- **Selective Application**: Only use CQRS for complex operations
- **Documentation**: Clear guidelines on when to use CQRS vs CRUD
- **Monitoring**: Track read model consistency and performance
- **Fallback**: Traditional queries available for debugging

## Implementation Guidelines

### When to Use CQRS:
✅ **Use CQRS for:**
- Complex business workflows (campaign approval, donation processing)
- Performance-critical queries (reports, dashboards)
- Operations requiring audit trails
- Multi-step operations with validation

❌ **Don't Use CQRS for:**
- Simple CRUD operations (user profile updates)
- Read-only reference data
- Operations without complex business logic

### Command Example:
```php
final class CreateCampaignCommand
{
    public function __construct(
        public readonly string $title,
        public readonly string $description,
        public readonly int $goalAmount,
        public readonly string $userId,
        public readonly array $metadata = []
    ) {}
}

final class CreateCampaignHandler
{
    public function __construct(
        private CampaignRepositoryInterface $repository,
        private EventDispatcherInterface $eventDispatcher
    ) {}

    public function handle(CreateCampaignCommand $command): Campaign
    {
        // Complex validation logic
        $this->validateCampaignData($command);

        // Create campaign entity
        $campaign = Campaign::create(
            title: $command->title,
            description: $command->description,
            goalAmount: $command->goalAmount,
            ownerId: $command->userId
        );

        // Persist to write model
        $this->repository->save($campaign);

        // Dispatch domain events
        $this->eventDispatcher->dispatch(
            new CampaignCreatedEvent($campaign->getId())
        );

        return $campaign;
    }
}
```

### Query Example:
```php
final class GetCampaignListQuery
{
    public function __construct(
        public readonly ?string $category = null,
        public readonly ?int $limit = 20,
        public readonly ?int $offset = 0,
        public readonly ?string $sortBy = 'created_at',
        public readonly ?string $sortOrder = 'desc'
    ) {}
}

final class GetCampaignListHandler
{
    public function __construct(
        private CampaignReadRepositoryInterface $readRepository
    ) {}

    public function handle(GetCampaignListQuery $query): CampaignListResult
    {
        // Use optimized read model
        return $this->readRepository->findByFilters([
            'category' => $query->category,
            'limit' => $query->limit,
            'offset' => $query->offset,
            'sort' => [$query->sortBy => $query->sortOrder]
        ]);
    }
}
```

## Read Model Synchronization

### Event-Driven Updates:
```php
final class CampaignCreatedEventHandler
{
    public function handle(CampaignCreatedEvent $event): void
    {
        // Update read model/materialized view
        $this->campaignReadRepository->updateFromDomainModel(
            $event->getCampaignId()
        );

        // Update search index
        $this->searchService->indexCampaign($event->getCampaignId());
    }
}
```

## Testing Strategy

```php
// Command Testing
public function testCreateCampaignCommand(): void
{
    $command = new CreateCampaignCommand(
        title: 'Test Campaign',
        description: 'Test Description',
        goalAmount: 10000,
        userId: 'user-123'
    );

    $campaign = $this->handler->handle($command);

    $this->assertEquals('Test Campaign', $campaign->getTitle());
    $this->assertTrue($campaign->isPending());
}

// Query Testing
public function testGetCampaignListQuery(): void
{
    $query = new GetCampaignListQuery(
        category: 'environment',
        limit: 10
    );

    $result = $this->handler->handle($query);

    $this->assertLessThanOrEqual(10, count($result->getCampaigns()));
    $this->assertTrue($result->hasNextPage());
}
```

## Related Decisions

- [ADR-001: Hexagonal Architecture](001-hexagonal-architecture.md) - Overall architecture pattern
- [ADR-004: Domain Driven Design](004-domain-driven-design.md) - Domain modeling approach

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved