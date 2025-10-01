# Hexagonal Architecture Implementation

## Overview

The ACME Corp CSR platform implements hexagonal architecture (ports and adapters pattern) with strict separation of concerns across Domain, Application, and Infrastructure layers. This architecture ensures business logic independence from external frameworks and technologies.

## Architecture Decision Rationale

### Why Hexagonal Architecture?

#### 1. Enterprise Scalability
```php
// Traditional Laravel Controller (Problematic)
class CampaignController extends Controller {
    public function store(Request $request) {
        // Business logic mixed with framework concerns
        $campaign = new Campaign();
        $campaign->name = $request->name;
        $campaign->goal = $request->goal;
        $campaign->save(); // Direct Eloquent dependency
        
        Mail::to($request->user())->send(new CampaignCreated($campaign));
        return response()->json($campaign);
    }
}

// Hexagonal Architecture Implementation (Superior)
final class CreateCampaignController {
    public function __invoke(
        CreateCampaignRequest $request,
        CommandBusInterface $commandBus
    ): JsonResponse {
        // Pure delegation to business logic
        $command = new CreateCampaignCommand(...$request->validated());
        $campaign = $commandBus->dispatch($command);
        
        return new JsonResponse(['id' => $campaign->getId()]);
    }
}
```

**Leadership Insight**: By isolating business logic from framework concerns, teams can work independently on domain logic without being blocked by framework constraints or updates.

#### 2. Technology Independence

The domain layer is completely free from framework dependencies:

```php
// modules/Campaign/Domain/Model/Campaign.php
declare(strict_types=1);

namespace Modules\Campaign\Domain\Model;

use Modules\Campaign\Domain\ValueObject\CampaignStatus;
use Modules\Campaign\Domain\ValueObject\Money;

final class Campaign {
    private readonly CampaignId $id;
    private string $name;
    private string $description;
    private Money $goal;
    private Money $raised;
    private CampaignStatus $status;
    
    // Pure business logic - no Laravel dependencies
    public function acceptDonation(Money $amount): DonationResult {
        if (!$this->status->isActive()) {
            return DonationResult::rejected('Campaign is not active');
        }
        
        if ($this->wouldExceedGoal($amount)) {
            return DonationResult::rejected('Donation would exceed campaign goal');
        }
        
        $this->raised = $this->raised->add($amount);
        
        if ($this->hasReachedGoal()) {
            $this->status = CampaignStatus::completed();
        }
        
        return DonationResult::accepted($amount);
    }
}
```

**Strategic Advantage**: This approach allows the business logic to be tested, understood, and modified without any external dependencies.

#### 3. Testing Excellence

Domain logic can be tested in pure isolation:

```php
// tests/Unit/Campaign/Domain/CampaignTest.php
class CampaignTest extends TestCase {
    public function test_campaign_accepts_valid_donation(): void {
        // Arrange - Pure domain objects, no database or framework
        $campaign = new Campaign(
            id: CampaignId::generate(),
            name: 'Clean Water Initiative',
            goal: Money::euros(1000),
            status: CampaignStatus::active()
        );
        
        // Act - Pure business logic
        $result = $campaign->acceptDonation(Money::euros(100));
        
        // Assert - Clear business rules validation
        $this->assertTrue($result->wasAccepted());
        $this->assertEquals(Money::euros(100), $campaign->getRaised());
    }
}
```

## CQRS Implementation Strategy

### Command-Query Separation

The platform implements strict CQRS to optimize read and write operations independently:

#### Commands (Write Operations)
```php
// modules/Campaign/Application/Command/CreateCampaignCommand.php
final readonly class CreateCampaignCommand implements CommandInterface {
    public function __construct(
        public string $name,
        public string $description,
        public float $goalAmount,
        public string $currency,
        public string $employeeId,
        public string $organizationId,
        public ?string $category = null
    ) {}
}

// modules/Campaign/Application/Command/CreateCampaignCommandHandler.php
final readonly class CreateCampaignCommandHandler implements CommandHandlerInterface {
    public function __construct(
        private CampaignRepositoryInterface $campaignRepository,
        private EventDispatcherInterface $eventDispatcher
    ) {}
    
    public function __invoke(CreateCampaignCommand $command): Campaign {
        $campaign = Campaign::create(
            name: $command->name,
            description: $command->description,
            goal: Money::from($command->goalAmount, $command->currency),
            employeeId: EmployeeId::fromString($command->employeeId),
            organizationId: OrganizationId::fromString($command->organizationId)
        );
        
        $this->campaignRepository->save($campaign);
        
        $this->eventDispatcher->dispatch(
            new CampaignCreatedEvent($campaign->getId())
        );
        
        return $campaign;
    }
}
```

#### Queries (Read Operations)
```php
// modules/Campaign/Application/Query/FindActiveCampaignsQuery.php
final readonly class FindActiveCampaignsQuery implements QueryInterface {
    public function __construct(
        public int $page = 1,
        public int $limit = 20,
        public ?string $category = null,
        public ?string $organizationId = null
    ) {}
}

// modules/Campaign/Application/Query/FindActiveCampaignsQueryHandler.php
final readonly class FindActiveCampaignsQueryHandler implements QueryHandlerInterface {
    public function __construct(
        private CampaignRepositoryInterface $campaignRepository
    ) {}
    
    public function __invoke(FindActiveCampaignsQuery $query): CampaignCollection {
        return $this->campaignRepository->findActive(
            page: $query->page,
            limit: $query->limit,
            category: $query->category,
            organizationId: $query->organizationId
        );
    }
}
```

### Strategic Benefits of CQRS

1. **Performance Optimization**: Read and write operations can be optimized independently
2. **Scalability**: Read replicas can handle queries while master handles commands
3. **Security**: Commands can have different authorization rules than queries
4. **Team Organization**: Different teams can work on read vs write operations

## Repository Pattern Implementation

### Interface Definition (Domain Layer)
```php
// modules/Campaign/Domain/Repository/CampaignRepositoryInterface.php
interface CampaignRepositoryInterface {
    public function save(Campaign $campaign): void;
    public function findById(CampaignId $id): ?Campaign;
    public function findActive(int $page, int $limit, ?string $category, ?string $organizationId): CampaignCollection;
    public function delete(CampaignId $id): void;
}
```

### Implementation (Infrastructure Layer)
```php
// modules/Campaign/Infrastructure/Laravel/Repository/CampaignEloquentRepository.php
final readonly class CampaignEloquentRepository implements CampaignRepositoryInterface {
    public function __construct(
        private CampaignEloquentModel $model
    ) {}
    
    public function save(Campaign $campaign): void {
        $eloquentModel = $this->model->firstOrNew(['id' => $campaign->getId()->toString()]);
        
        $eloquentModel->fill([
            'id' => $campaign->getId()->toString(),
            'name' => $campaign->getName(),
            'description' => $campaign->getDescription(),
            'goal_amount' => $campaign->getGoal()->getAmount(),
            'goal_currency' => $campaign->getGoal()->getCurrency(),
            'raised_amount' => $campaign->getRaised()->getAmount(),
            'status' => $campaign->getStatus()->toString(),
            'employee_id' => $campaign->getEmployeeId()->toString(),
            'organization_id' => $campaign->getOrganizationId()->toString(),
        ]);
        
        $eloquentModel->save();
    }
    
    public function findById(CampaignId $id): ?Campaign {
        $eloquentModel = $this->model->find($id->toString());
        
        if (!$eloquentModel) {
            return null;
        }
        
        return $this->toDomainModel($eloquentModel);
    }
    
    private function toDomainModel(CampaignEloquentModel $model): Campaign {
        return Campaign::reconstruct(
            id: CampaignId::fromString($model->id),
            name: $model->name,
            description: $model->description,
            goal: Money::from($model->goal_amount, $model->goal_currency),
            raised: Money::from($model->raised_amount, $model->goal_currency),
            status: CampaignStatus::fromString($model->status),
            employeeId: EmployeeId::fromString($model->employee_id),
            organizationId: OrganizationId::fromString($model->organization_id),
            createdAt: $model->created_at,
            updatedAt: $model->updated_at
        );
    }
}
```

## Domain Events Strategy

### Event-Driven Architecture
```php
// modules/Campaign/Domain/Event/CampaignGoalReachedEvent.php
final readonly class CampaignGoalReachedEvent implements DomainEventInterface {
    public function __construct(
        public CampaignId $campaignId,
        public Money $finalAmount,
        public DateTimeImmutable $completedAt
    ) {}
    
    public function getAggregateId(): string {
        return $this->campaignId->toString();
    }
    
    public function getEventName(): string {
        return 'campaign.goal_reached';
    }
}

// Event Handler
final readonly class SendCampaignCompletionNotificationHandler {
    public function __construct(
        private NotificationServiceInterface $notificationService,
        private CampaignRepositoryInterface $campaignRepository
    ) {}
    
    public function __invoke(CampaignGoalReachedEvent $event): void {
        $campaign = $this->campaignRepository->findById($event->campaignId);
        
        if (!$campaign) {
            return; // Campaign deleted or not found
        }
        
        $this->notificationService->notifyCampaignCompletion(
            campaign: $campaign,
            completedAt: $event->completedAt
        );
    }
}
```

## Value Objects for Domain Integrity

### Money Value Object
```php
// modules/Shared/Domain/ValueObject/Money.php
final readonly class Money {
    private function __construct(
        private int $amount, // Store in cents to avoid floating point issues
        private string $currency
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
        
        if (!in_array($currency, ['EUR', 'USD', 'GBP'], true)) {
            throw new InvalidArgumentException('Unsupported currency');
        }
    }
    
    public static function euros(float $amount): self {
        return new self((int) round($amount * 100), 'EUR');
    }
    
    public static function from(float $amount, string $currency): self {
        return new self((int) round($amount * 100), $currency);
    }
    
    public function add(self $other): self {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('Cannot add different currencies');
        }
        
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    public function getAmount(): float {
        return $this->amount / 100;
    }
    
    public function getCurrency(): string {
        return $this->currency;
    }
    
    public function isGreaterThan(self $other): bool {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('Cannot compare different currencies');
        }
        
        return $this->amount > $other->amount;
    }
}
```

## Dependency Injection Strategy

### Service Provider Configuration
```php
// modules/Campaign/Infrastructure/Laravel/Provider/CampaignServiceProvider.php
final class CampaignServiceProvider extends ServiceProvider {
    public function register(): void {
        // Bind repository interface to implementation
        $this->app->bind(
            CampaignRepositoryInterface::class,
            CampaignEloquentRepository::class
        );
        
        // Register command handlers
        $this->app->bind(
            CreateCampaignCommandHandler::class,
            fn (Application $app) => new CreateCampaignCommandHandler(
                $app->make(CampaignRepositoryInterface::class),
                $app->make(EventDispatcherInterface::class)
            )
        );
        
        // Register query handlers
        $this->app->bind(
            FindActiveCampaignsQueryHandler::class,
            fn (Application $app) => new FindActiveCampaignsQueryHandler(
                $app->make(CampaignRepositoryInterface::class)
            )
        );
    }
    
    public function boot(): void {
        // Register event listeners
        Event::listen(
            CampaignGoalReachedEvent::class,
            SendCampaignCompletionNotificationHandler::class
        );
    }
}
```

## Single Action Controllers

Every controller follows the single responsibility principle:

```php
// modules/Campaign/Infrastructure/Laravel/Controllers/ShowCampaignController.php
final readonly class ShowCampaignController {
    public function __construct(
        private QueryBusInterface $queryBus
    ) {}
    
    public function __invoke(string $campaignId): JsonResponse {
        $query = new FindCampaignByIdQuery($campaignId);
        $campaign = $this->queryBus->dispatch($query);
        
        if (!$campaign) {
            throw new ModelNotFoundException('Campaign not found');
        }
        
        return new JsonResponse([
            'data' => new CampaignResource($campaign)
        ]);
    }
}
```

## Architectural Boundaries Enforcement

### Deptrac Configuration
```yaml
# deptrac.yaml
deptrac:
  paths:
    - ./modules/
  layers:
    - name: Domain
      collectors:
        - type: directory
          regex: modules/.*/Domain/.*
    - name: Application  
      collectors:
        - type: directory
          regex: modules/.*/Application/.*
    - name: Infrastructure
      collectors:
        - type: directory
          regex: modules/.*/Infrastructure/.*
  ruleset:
    Domain: [] # Domain depends on nothing
    Application: [Domain] # Application can use Domain
    Infrastructure: [Application, Domain] # Infrastructure can use both
```

### Continuous Architectural Validation
```bash
# Automated boundary checking in CI/CD
vendor/bin/deptrac analyse --fail-on-uncovered
```

## Generator Commands for Consistency

The platform includes custom generators to ensure architectural consistency:

```bash
# Generate complete domain structure
php artisan add:hex:domain NewFeature

# Generate specific components
php artisan add:hex:command CreateSomething --domain=NewFeature
php artisan add:hex:query FindSomething --domain=NewFeature
php artisan add:hex:repository Something --domain=NewFeature
```

These generators ensure every new feature follows the established patterns and reduces the learning curve for new team members.

## Performance Implications

### Query Optimization
```php
// Repository implementation with performance considerations
final readonly class CampaignEloquentRepository implements CampaignRepositoryInterface {
    public function findActive(int $page, int $limit, ?string $category, ?string $organizationId): CampaignCollection {
        $query = $this->model
            ->where('status', CampaignStatus::ACTIVE)
            ->with(['organization', 'employee']) // Eager loading to prevent N+1
            ->when($category, fn ($q) => $q->where('category', $category))
            ->when($organizationId, fn ($q) => $q->where('organization_id', $organizationId))
            ->orderBy('created_at', 'desc');
            
        $results = $query->paginate($limit, ['*'], 'page', $page);
        
        return new CampaignCollection(
            $results->items(),
            $results->total(),
            $results->currentPage(),
            $results->perPage()
        );
    }
}
```

## Migration Strategy

For teams adopting hexagonal architecture, the migration path is:

1. **Extract Domain Models**: Move business logic from Eloquent models to pure domain models
2. **Implement Repository Pattern**: Create interfaces in domain, implementations in infrastructure
3. **Refactor Controllers**: Convert to single-action controllers that delegate to command/query handlers
4. **Add CQRS**: Separate read and write operations
5. **Implement Domain Events**: Add event-driven communication between bounded contexts
6. **Enforce Boundaries**: Use Deptrac to prevent architectural violations

## Strategic Benefits

### For Development Teams
- **Clear Responsibility**: Each layer has well-defined responsibilities
- **Independent Work**: Teams can work on different layers without conflicts
- **Easy Testing**: Domain logic can be tested without external dependencies
- **Technology Migration**: Framework changes don't affect business logic

### For Business Stakeholders
- **Faster Feature Development**: Established patterns accelerate development
- **Reduced Technical Debt**: Clean architecture prevents accumulation of debt
- **Better Quality**: Separation of concerns leads to higher code quality
- **Future-Proof**: Architecture adapts to changing business requirements

This hexagonal architecture implementation represents technical leadership through strategic architectural decisions that prioritize long-term success over short-term convenience, demonstrating the planning and foresight expected in enterprise software development.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved