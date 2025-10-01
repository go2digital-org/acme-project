# ADR-004: Domain Driven Design Principles and Bounded Contexts

## Status

Accepted

## Context

The ACME Corp CSR Platform is a complex enterprise application spanning multiple business domains:
- **Campaign Management**: Creating, approving, and managing charitable campaigns
- **User Management**: Employee authentication, profiles, and permissions
- **Donation Processing**: Payment handling, matching funds, and receipts
- **Notification System**: Email, SMS, and in-app notifications
- **Reporting & Analytics**: Financial reports, impact metrics, and dashboards

As the platform grows, we need to:
- Manage complexity through clear domain boundaries
- Enable independent team development
- Reduce coupling between different business areas
- Establish a common language between business and development teams

We evaluated several domain modeling approaches:
1. **Monolithic Domain Model**: Single large model, simple but becomes unwieldy
2. **Domain Driven Design**: Rich domain models with clear boundaries
3. **Transaction Script**: Procedural approach, fast but doesn't scale
4. **Data-Driven Design**: Database-centric, lacks business logic encapsulation

## Decision

We will apply **Domain Driven Design (DDD)** principles with clearly defined **Bounded Contexts** for the ACME Corp CSR Platform.

### Core DDD Principles Applied:
1. **Ubiquitous Language**: Common vocabulary between business and developers
2. **Bounded Contexts**: Clear boundaries between different business domains
3. **Domain Models**: Rich entities with business logic encapsulation
4. **Aggregates**: Consistency boundaries for related entities
5. **Domain Services**: Business logic that doesn't belong to a single entity
6. **Value Objects**: Immutable objects representing descriptive concepts

### Bounded Contexts Identified:
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Campaign      │  │      User       │  │   Donation      │
│   Context       │  │    Context      │  │   Context       │
│                 │  │                 │  │                 │
│ • Campaign      │  │ • User          │  │ • Donation      │
│ • Category      │  │ • Profile       │  │ • Payment       │
│ • Approval      │  │ • Permission    │  │ • Receipt       │
│ • Goal          │  │ • Department    │  │ • Matching      │
└─────────────────┘  └─────────────────┘  └─────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Notification    │  │   Reporting     │  │  Integration    │
│   Context       │  │   Context       │  │   Context       │
│                 │  │                 │  │                 │
│ • Notification  │  │ • Report        │  │ • PaymentGateway│
│ • Template      │  │ • Metric        │  │ • EmailService  │
│ • Channel       │  │ • Dashboard     │  │ • SearchEngine  │
│ • Preference    │  │ • Export        │  │ • FileStorage   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## Consequences

### Positive:
- **Clear Boundaries**: Each context has well-defined responsibilities
- **Team Autonomy**: Different teams can work on different contexts independently
- **Scalability**: Contexts can evolve at different rates
- **Business Alignment**: Code structure reflects business organization
- **Maintainability**: Changes in one context don't affect others
- **Testing**: Each context can be tested in isolation
- **Rich Domain Models**: Business logic is encapsulated where it belongs

### Negative:
- **Initial Complexity**: More upfront design and modeling effort
- **Learning Curve**: Team needs to understand DDD concepts
- **Integration Complexity**: Cross-context communication requires careful design
- **Over-Engineering Risk**: Simple operations might become unnecessarily complex

### Risk Mitigation:
- **Incremental Adoption**: Start with core contexts and expand gradually
- **Clear Documentation**: Maintain context maps and domain glossaries
- **Regular Reviews**: Periodic assessment of context boundaries
- **Anti-Corruption Layers**: Protect domain models from external influences

## Domain Model Examples

### Campaign Context - Rich Aggregate:
```php
final class Campaign
{
    private CampaignId $id;
    private string $title;
    private string $description;
    private Money $goalAmount;
    private Money $currentAmount;
    private CampaignStatus $status;
    private UserId $ownerId;
    private \DateTimeImmutable $createdAt;
    private ?DateTimeImmutable $approvedAt;

    public function approve(UserId $approverId): void
    {
        if (!$this->status->isPending()) {
            throw new CampaignAlreadyProcessedException($this->id);
        }

        if ($this->goalAmount->isLessThan(Money::fromAmount(100))) {
            throw new InsufficientGoalAmountException($this->id);
        }

        $this->status = CampaignStatus::approved();
        $this->approvedAt = new \DateTimeImmutable();

        // Domain event
        $this->recordEvent(new CampaignApprovedEvent($this->id, $approverId));
    }

    public function receiveDonation(Money $amount, UserId $donorId): Donation
    {
        if (!$this->status->isActive()) {
            throw new CampaignNotActiveException($this->id);
        }

        if ($this->isGoalReached()) {
            throw new CampaignGoalReachedException($this->id);
        }

        $donation = Donation::create($this->id, $amount, $donorId);
        $this->currentAmount = $this->currentAmount->add($amount);

        $this->recordEvent(new DonationReceivedEvent($donation->getId()));

        return $donation;
    }
}
```

### Value Objects:
```php
final class Money
{
    public function __construct(
        private readonly int $amount, // in cents
        private readonly Currency $currency
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException('Amount cannot be negative');
        }
    }

    public static function fromAmount(float $amount, Currency $currency = Currency::USD): self
    {
        return new self((int) round($amount * 100), $currency);
    }

    public function add(Money $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function isLessThan(Money $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount < $other->amount;
    }
}

enum CampaignStatus: string
{
    case DRAFT = 'draft';
    case PENDING = 'pending';
    case APPROVED = 'approved';
    case ACTIVE = 'active';
    case COMPLETED = 'completed';
    case CANCELLED = 'cancelled';

    public function isPending(): bool
    {
        return $this === self::PENDING;
    }

    public function isActive(): bool
    {
        return $this === self::ACTIVE;
    }
}
```

### Domain Services:
```php
final class CampaignApprovalService
{
    public function __construct(
        private CampaignRepositoryInterface $repository,
        private UserRepositoryInterface $userRepository,
        private ComplianceCheckService $complianceService
    ) {}

    public function approve(CampaignId $campaignId, UserId $approverId): void
    {
        $campaign = $this->repository->findById($campaignId);
        $approver = $this->userRepository->findById($approverId);

        if (!$approver->hasPermission(Permission::APPROVE_CAMPAIGNS)) {
            throw new InsufficientPermissionsException($approverId);
        }

        if (!$this->complianceService->passesComplianceCheck($campaign)) {
            throw new ComplianceCheckFailedException($campaignId);
        }

        $campaign->approve($approverId);
        $this->repository->save($campaign);
    }
}
```

## Context Integration Patterns

### Domain Events for Cross-Context Communication:
```php
// Published by Campaign Context
final class CampaignApprovedEvent
{
    public function __construct(
        public readonly CampaignId $campaignId,
        public readonly UserId $approverId,
        public readonly \DateTimeImmutable $approvedAt
    ) {}
}

// Consumed by Notification Context
final class CampaignApprovedEventHandler
{
    public function handle(CampaignApprovedEvent $event): void
    {
        $this->notificationService->sendCampaignApprovalNotification(
            $event->campaignId,
            $event->approverId
        );
    }
}
```

### Anti-Corruption Layer for External Services:
```php
interface PaymentGatewayInterface
{
    public function processPayment(PaymentRequest $request): PaymentResult;
}

final class StripePaymentGateway implements PaymentGatewayInterface
{
    public function processPayment(PaymentRequest $request): PaymentResult
    {
        // Convert domain request to Stripe format
        $stripeRequest = $this->convertToStripeFormat($request);

        // Call Stripe API
        $stripeResponse = $this->stripeClient->charges->create($stripeRequest);

        // Convert Stripe response to domain format
        return $this->convertToDomainResult($stripeResponse);
    }
}
```

## Ubiquitous Language

### Campaign Context Vocabulary:
- **Campaign**: A charitable initiative with a funding goal
- **Goal Amount**: The target funding amount for a campaign
- **Approval**: The process of validating and activating a campaign
- **Donation**: A monetary contribution to a campaign
- **Matching**: Corporate matching of employee donations

### User Context Vocabulary:
- **User**: An authenticated person using the system
- **Profile**: User's personal information and preferences
- **Department**: Organizational unit for grouping users
- **Permission**: Authorization to perform specific actions

## Testing Strategy

```php
// Domain Model Testing
public function testCampaignApproval(): void
{
    $campaign = CampaignTestBuilder::aPendingCampaign()
        ->withGoalAmount(Money::fromAmount(1000))
        ->build();

    $approverId = UserId::generate();
    $campaign->approve($approverId);

    $this->assertTrue($campaign->getStatus()->isApproved());
    $this->assertNotNull($campaign->getApprovedAt());
}

// Domain Service Testing
public function testCampaignApprovalService(): void
{
    $campaign = $this->givenAPendingCampaign();
    $approver = $this->givenAnApproverUser();

    $this->complianceService
        ->shouldReceive('passesComplianceCheck')
        ->with($campaign)
        ->andReturn(true);

    $this->service->approve($campaign->getId(), $approver->getId());

    $this->assertTrue($campaign->getStatus()->isApproved());
}
```

## Related Decisions

- [ADR-001: Hexagonal Architecture](001-hexagonal-architecture.md) - Technical architecture pattern
- [ADR-003: CQRS Pattern](003-cqrs-pattern.md) - Application layer organization
- [ADR-005: Employee to User Migration](005-employee-to-user-migration.md) - Domain model evolution

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved