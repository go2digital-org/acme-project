# Developer Onboarding Guide

Welcome to the ACME Corp CSR Platform! This comprehensive guide will help you get up to speed with our enterprise-grade Corporate Social Responsibility platform built with Hexagonal Architecture and Domain-Driven Design principles.

## Project Overview

### What We're Building
The ACME Corp CSR Platform is an enterprise-grade Corporate Social Responsibility platform designed to handle 20,000+ concurrent users across global operations. It enables organizations to:

- Create and manage fundraising campaigns
- Process donations with multiple payment gateways
- Track user contributions and generate reports
- Manage organizational departments and hierarchies
- Provide real-time notifications and updates

### Technology Stack
- **Backend**: PHP 8.4, Laravel 11.x
- **API**: API Platform 4.x with OpenAPI documentation
- **Database**: MySQL 8.0 with Redis caching
- **Search**: Meilisearch for full-text search
- **Frontend**: Livewire 3.x, Filament 4.x Admin Panel
- **Queue**: Redis-based job processing
- **Testing**: Pest 4 with Browser Plugin
- **Architecture**: Hexagonal Architecture + Domain-Driven Design + CQRS

### Key Metrics
- **Tests**: 3,814 tests with 96% pass rate
- **Coverage**: 90%+ code coverage requirement
- **Code Quality**: PHPStan Level 8, Laravel Pint, Deptrac architecture validation
- **Performance**: Sub-200ms response times, horizontal scaling capable

## Development Environment Setup

### Prerequisites
Before starting, ensure you have:
- PHP 8.4+ with required extensions (BCMath, Redis, etc.)
- MySQL 8.0+
- Redis 6.0+
- Node.js 18+ and NPM
- Composer 2.0+
- Git
- Docker & Docker Compose (for Docker setup)

### Environment Choice: Docker vs Local

We support both Docker and local development environments. **Docker is recommended** for new developers.

#### Option 1: Docker Environment (Recommended)

**Advantages:**
- Consistent environment across all developers
- No need to install PHP, MySQL, Redis locally
- Pre-configured services (Meilisearch, Mailpit)
- Production-like environment

```bash
# Clone the repository
git clone https://github.com/your-username/acme-corp-optimy.git
cd acme-corp-csr-platform

# Start all services with Docker Compose
docker-compose --env-file .env.docker up -d

# Install dependencies
docker-compose exec app composer install
docker-compose exec app npm install
docker-compose exec app npm run build

# Run migrations and seed data
docker-compose exec app php artisan migrate --seed

# Import search indexes
docker-compose exec app php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'
```

**Access Points:**
- Application: http://localhost (or http://app.acme-corp-optimy.orb.local/ with OrbStack)
- Mailpit UI: http://localhost:8025
- Meilisearch: http://localhost:7700

#### Option 2: Local Development Environment

**When to use:**
- You prefer working with local PHP/MySQL
- You need to debug with XDebug
- You want faster file system access

```bash
# Clone and install
git clone https://github.com/your-username/acme-corp-optimy.git
cd acme-corp-csr-platform
composer install --optimize-autoloader
npm install

# Configure environment
cp .env.example .env
php artisan key:generate

# Setup database
mysql -u root -p -e "CREATE DATABASE acme_corp_csr CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
php artisan migrate --seed

# Start development server
php artisan serve
```

**Access:** http://localhost:8000

### Environment Files
- `.env` - Local development (127.0.0.1 addresses)
- `.env.docker` - Docker environment (service names: mysql, redis, etc.)
- `.env.testing` - Test environment configuration

## Architecture Deep Dive

### Hexagonal Architecture Overview

Our platform implements **Hexagonal Architecture** (also known as Ports and Adapters) to achieve:
- **Technology Independence**: Business logic isolated from frameworks
- **Testability**: Pure domain logic tested without external dependencies
- **Maintainability**: Clear separation of concerns
- **Team Productivity**: Multiple teams can work independently

```
Architecture Layers:

Infrastructure Layer (Outermost)
├── Controllers (API endpoints)
├── Repositories (Data access)
├── External APIs (Payment gateways)
└── Queue Jobs (Background processing)
         ↓
Application Layer (Middle)
├── Command Handlers (Write operations)
├── Query Handlers (Read operations)
├── Read Models (Optimized queries)
└── Event Handlers (Cross-cutting)
         ↓
Domain Layer (Core)
├── Domain Models (Business rules)
├── Value Objects (Type safety)
├── Specifications (Business rules)
└── Domain Events (State changes)
```

### Key Architectural Principles

#### 1. Domain-Driven Design (DDD)
- **Rich Domain Models**: Business logic encapsulated in domain objects
- **Ubiquitous Language**: Common vocabulary between developers and business
- **Bounded Contexts**: Clear module boundaries

#### 2. CQRS (Command Query Responsibility Segregation)
- **Commands**: Write operations that change state
- **Queries**: Read operations optimized for display
- **Separate Models**: Different models for reads and writes

#### 3. Event-Driven Architecture
- **Domain Events**: Decouple modules through events
- **Event Handlers**: React to state changes
- **Eventual Consistency**: Loose coupling between modules

## Module Structure

Each module follows a consistent structure:

```
modules/[ModuleName]/
├── Domain/                    # Core business logic
│   ├── Model/                # Rich domain models
│   ├── ValueObject/          # Immutable value objects
│   ├── Repository/           # Repository interfaces
│   ├── Specification/        # Business rule specifications
│   ├── Exception/            # Domain-specific exceptions
│   └── Event/                # Domain events
├── Application/              # Use case orchestration
│   ├── Command/              # CQRS Commands and Handlers
│   ├── Query/                # CQRS Queries and Handlers
│   ├── ReadModel/            # Optimized read models
│   └── Service/              # Application services
└── Infrastructure/           # External adapters
    ├── Laravel/              # Laravel-specific implementations
    │   ├── Controllers/      # Single-action controllers
    │   ├── Models/           # Eloquent models (persistence)
    │   ├── Repository/       # Repository implementations
    │   ├── Migration/        # Database migrations
    │   ├── Factory/          # Model factories
    │   ├── Seeder/           # Database seeders
    │   └── Provider/         # Service providers
    ├── ApiPlatform/          # API Platform adapters
    │   ├── Handler/          # Request processors
    │   └── Resource/         # API resource definitions
    └── Filament/             # Admin interface adapters
        ├── Pages/            # Custom admin pages
        └── Resources/        # Admin resource definitions
```

### Module Examples

Let's look at the **Campaign** module as an example:

#### Domain Layer (Pure Business Logic)
```php
// modules/Campaign/Domain/Model/Campaign.php
final class Campaign {
    private readonly CampaignId $id;
    private string $name;
    private Money $goal;
    private Money $raised;
    private CampaignStatus $status;

    // Pure business logic - no framework dependencies
    public function acceptDonation(Money $amount): DonationResult {
        if (!$this->status->isActive()) {
            return DonationResult::rejected('Campaign is not active');
        }

        if ($this->wouldExceedGoal($amount)) {
            return DonationResult::rejected('Donation would exceed goal');
        }

        $this->raised = $this->raised->add($amount);

        if ($this->hasReachedGoal()) {
            $this->status = CampaignStatus::completed();
        }

        return DonationResult::accepted($amount);
    }
}
```

#### Application Layer (Use Cases)
```php
// modules/Campaign/Application/Command/CreateCampaignCommandHandler.php
final readonly class CreateCampaignCommandHandler {
    public function __construct(
        private CampaignRepositoryInterface $repository,
        private EventDispatcherInterface $eventDispatcher
    ) {}

    public function handle(CreateCampaignCommand $command): Campaign {
        // Orchestrate domain objects
        $campaign = Campaign::create(
            name: $command->name,
            goal: Money::fromString($command->goalAmount, $command->currency),
            userId: UserId::fromString($command->userId)
        );

        $this->repository->save($campaign);

        // Dispatch domain event
        $this->eventDispatcher->dispatch(
            new CampaignCreated($campaign->getId())
        );

        return $campaign;
    }
}
```

#### Infrastructure Layer (Framework Integration)
```php
// modules/Campaign/Infrastructure/Laravel/Controllers/CreateCampaignController.php
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

## Adding New Features

### The Hexagonal Way

When adding new features, follow this process:

#### 1. Start with Domain (Inside-Out)
```bash
# Use our interactive menu for structured creation
php artisan hex:menu

# Or create individual components
php artisan hex:add:model NewFeature
php artisan hex:add:repository NewFeature
```

#### 2. Domain Layer First
- Create domain models with business rules
- Add value objects for type safety
- Define repository interfaces (contracts)
- Write domain specifications

#### 3. Application Layer Second
- Create commands for write operations
- Create queries for read operations
- Implement command/query handlers
- Add read models for optimized queries

#### 4. Infrastructure Layer Last
- Implement repository with Eloquent
- Create migrations and factories
- Add API Platform resources
- Build controllers for endpoints

### Example: Adding a "Donation" Feature

#### Step 1: Domain Model
```php
// modules/Donation/Domain/Model/Donation.php
final class Donation {
    public function __construct(
        private readonly DonationId $id,
        private readonly CampaignId $campaignId,
        private readonly UserId $userId,
        private readonly Money $amount,
        private readonly PaymentMethod $paymentMethod,
        private DonationStatus $status,
        private readonly \DateTimeImmutable $createdAt
    ) {}

    public function process(PaymentGateway $gateway): PaymentResult {
        // Business logic for processing donation
        if ($this->amount->isZero()) {
            throw new InvalidDonationAmountException();
        }

        $result = $gateway->charge($this->amount, $this->paymentMethod);

        if ($result->isSuccessful()) {
            $this->status = DonationStatus::completed();
        } else {
            $this->status = DonationStatus::failed();
        }

        return $result;
    }
}
```

#### Step 2: Application Command
```php
// modules/Donation/Application/Command/ProcessDonationCommand.php
final readonly class ProcessDonationCommand {
    public function __construct(
        public string $donationId,
        public string $campaignId,
        public string $userId,
        public string $amount,
        public string $currency,
        public string $paymentMethod
    ) {}
}

// Handler
final readonly class ProcessDonationCommandHandler {
    public function handle(ProcessDonationCommand $command): Donation {
        // Orchestrate domain objects
        $donation = Donation::create(/* ... */);
        $result = $donation->process($this->paymentGateway);

        if ($result->isSuccessful()) {
            $this->repository->save($donation);
            $this->eventDispatcher->dispatch(new DonationProcessed($donation));
        }

        return $donation;
    }
}
```

#### Step 3: Infrastructure Controller
```php
// modules/Donation/Infrastructure/Laravel/Controllers/ProcessDonationController.php
final class ProcessDonationController {
    public function __invoke(
        ProcessDonationRequest $request,
        CommandBusInterface $commandBus
    ): JsonResponse {
        $command = new ProcessDonationCommand(...$request->validated());
        $donation = $commandBus->dispatch($command);

        return new JsonResponse([
            'donation_id' => $donation->getId(),
            'status' => $donation->getStatus()
        ]);
    }
}
```

## Testing Guidelines

### Testing Strategy by Layer

Our testing follows the architecture layers:

```
Test Structure:

tests/Unit/                    # Domain Layer Tests
├── [Module]/Domain/
│   ├── Model/                # Domain model tests
│   └── ValueObject/          # Value object tests
└── [Module]/Application/     # Command/Query handler tests

tests/Integration/             # Infrastructure Layer Tests
└── [Module]/Infrastructure/
    ├── Repository/           # Repository implementation tests
    └── External/             # External service tests

tests/Feature/                 # API Tests (End-to-end)
└── Api/[Module]/             # API endpoint tests

tests/Browser/                 # E2E Tests (User journeys)
└── [Module]/                 # Browser automation tests

tests/Architecture/            # Architectural Boundary Tests
├── ModuleBoundariesTest.php
└── HexagonalArchitectureTest.php
```

### Testing Commands

```bash
# Quick test commands
./vendor/bin/pest --testsuite=Unit          # Pure domain logic (fast)
./vendor/bin/pest --testsuite=Integration   # Infrastructure tests
./vendor/bin/pest --testsuite=Feature       # API endpoint tests
./vendor/bin/pest tests/Browser/             # Browser tests

# Parallel execution for speed
./vendor/bin/pest --parallel

# Coverage analysis
./vendor/bin/pest --coverage --min=90

# Architecture validation
./vendor/bin/pest tests/Architecture/

# Specific tests
./vendor/bin/pest tests/Unit/Campaign/Domain/Model/CampaignTest.php
./vendor/bin/pest --filter="can accept valid donation"
```

### Test Examples by Layer

#### Unit Tests (Domain Layer)
```php
// tests/Unit/Campaign/Domain/Model/CampaignTest.php
test('campaign accepts valid donation', function () {
    // Pure domain testing - no database, no framework
    $campaign = Campaign::create(
        name: 'Clean Water Initiative',
        goal: Money::euros(1000),
        userId: UserId::generate()
    );

    $result = $campaign->acceptDonation(Money::euros(100));

    expect($result->isAccepted())->toBeTrue();
    expect($campaign->getRaised())->toEqual(Money::euros(100));
});

test('campaign rejects donation when inactive', function () {
    $campaign = Campaign::create(/* ... */);
    $campaign->deactivate();

    $result = $campaign->acceptDonation(Money::euros(100));

    expect($result->isRejected())->toBeTrue();
    expect($result->getReason())->toBe('Campaign is not active');
});
```

#### Integration Tests (Infrastructure Layer)
```php
// tests/Integration/Campaign/Infrastructure/Repository/CampaignRepositoryTest.php
test('saves and retrieves campaign from database', function () {
    // Test database integration
    $campaign = Campaign::create(/* ... */);

    $this->campaignRepository->save($campaign);
    $retrieved = $this->campaignRepository->findById($campaign->getId());

    expect($retrieved)->not->toBeNull();
    expect($retrieved->getName())->toBe($campaign->getName());
    expect($retrieved->getGoal())->toEqual($campaign->getGoal());
});
```

#### Feature Tests (API Layer)
```php
// tests/Feature/Api/Campaign/CreateCampaignTest.php
test('creates campaign with valid data', function () {
    // Test complete API workflow
    $response = $this->postJson('/api/campaigns', [
        'name' => 'Test Campaign',
        'description' => 'Test Description',
        'goal' => ['amount' => 1000, 'currency' => 'EUR'],
        'organization_id' => 'org-123'
    ], [
        'Authorization' => 'Bearer ' . $this->getAuthToken()
    ]);

    $response->assertStatus(201);
    $response->assertJsonStructure(['data' => ['id', 'name', 'status']]);

    // Verify in database
    $this->assertDatabaseHas('campaigns', [
        'name' => 'Test Campaign',
        'status' => 'draft'
    ]);
});
```

#### Browser Tests (E2E)
```php
// tests/Browser/Campaign/CampaignManagementTest.php
test('user can create campaign through UI', function () {
    // Test complete user workflow
    browse(function (Browser $browser) {
        $browser->loginAs($this->user)
                ->visit('/campaigns/create')
                ->type('name', 'My New Campaign')
                ->type('description', 'Campaign description')
                ->type('goal_amount', '1000')
                ->select('currency', 'EUR')
                ->press('Create Campaign')
                ->assertPathIs('/campaigns')
                ->assertSee('Campaign created successfully')
                ->assertSee('My New Campaign');
    });
});
```

#### Architecture Tests
```php
// tests/Architecture/HexagonalArchitectureTest.php
test('domain layer has no framework dependencies', function () {
    // Ensure architectural boundaries
    Arch::expect('Modules\\*\\Domain')
        ->not->toUse(['Illuminate\\*', 'Laravel\\*', 'Filament\\*']);
});

test('infrastructure depends on domain but not vice versa', function () {
    Arch::expect('Modules\\*\\Infrastructure')
        ->toUse('Modules\\*\\Domain');

    Arch::expect('Modules\\*\\Domain')
        ->not->toUse('Modules\\*\\Infrastructure');
});

test('controllers are single action', function () {
    Arch::expect('Modules\\*\\Infrastructure\\Laravel\\Controllers\\*')
        ->toHaveMethod('__invoke')
        ->toExtendNothing(); // No base controller inheritance
});
```

### Writing Good Tests

#### Domain Tests (Unit)
- **Fast**: No database, no external services
- **Isolated**: Pure business logic testing
- **Focused**: One business rule per test
- **Readable**: Clear Given-When-Then structure

#### Infrastructure Tests (Integration)
- **Real Dependencies**: Actual database, Redis, etc.
- **Rollback**: Use transactions for cleanup
- **Focused**: Test one integration at a time
- **Realistic Data**: Use factories for test data

#### API Tests (Feature)
- **Complete Workflow**: Test entire request/response cycle
- **Authentication**: Test with real JWT tokens
- **Validation**: Test both success and error cases
- **Database State**: Verify database changes

## Code Quality Standards

### Quality Tools Stack

```bash
# Static Analysis (PHPStan Level 8)
./vendor/bin/phpstan analyse

# Code Style (Laravel Pint)
./vendor/bin/pint                    # Fix code style
./vendor/bin/pint --test             # Check without fixing

# Automated Refactoring (Rector)
./vendor/bin/rector process          # Apply refactoring rules
./vendor/bin/rector process --dry-run # Preview changes

# Architecture Validation (Deptrac)
./vendor/bin/deptrac analyse         # Check layer boundaries

# All Quality Checks (GrumPHP)
./vendor/bin/grumphp run             # Run all checks
```

### Quality Standards

#### 1. PHPStan Level 8
- **No Mixed Types**: Everything must be properly typed
- **No Unknown Properties**: All properties declared
- **No Dead Code**: Unused code is detected
- **Strict Type Checking**: Type safety enforced

#### 2. Laravel Pint (PSR-12)
- **Consistent Formatting**: Automated code formatting
- **Modern PHP**: Uses latest PHP syntax features
- **Laravel Conventions**: Follows Laravel coding standards

#### 3. Rector Rules
- **Modernization**: Upgrades to latest PHP features
- **Performance**: Optimizes code patterns
- **Consistency**: Applies consistent patterns
- **Dead Code Removal**: Removes unused imports/variables

#### 4. Deptrac Architecture
- **Layer Boundaries**: Prevents architectural violations
- **Module Isolation**: Ensures modules don't leak
- **Dependency Direction**: Enforces dependency rules

### Pre-Commit Workflow

```bash
# Before committing, run:
./vendor/bin/grumphp run            # All quality checks
./vendor/bin/pest                   # All tests

# Or use the make commands:
make quality                        # Quality checks only
make test                          # Tests only
make all                           # Quality + tests
```

### Code Review Checklist

#### Architecture
- [ ] Follows hexagonal architecture principles
- [ ] Domain logic is framework-independent
- [ ] Proper layer separation (Domain → Application → Infrastructure)
- [ ] No architectural violations detected by Deptrac

#### Code Quality
- [ ] PHPStan Level 8 compliance
- [ ] All tests passing (Unit, Integration, Feature)
- [ ] Code style follows Laravel Pint rules
- [ ] Rector rules applied

#### Business Logic
- [ ] Business rules implemented in domain layer
- [ ] Domain models are rich and expressive
- [ ] Value objects used for type safety
- [ ] Proper exception handling

#### Testing
- [ ] Unit tests for domain logic
- [ ] Integration tests for infrastructure
- [ ] Feature tests for API endpoints
- [ ] Test coverage >= 90%

## Common Development Tasks

### 1. Adding a New Module

```bash
# Interactive approach (recommended)
php artisan hex:menu
# Select: "Create complete domain structure"

# Manual approach
php artisan hex:add:structure NewModule
php artisan hex:add:model NewModule
php artisan hex:add:repository NewModule
php artisan hex:add:repository-eloquent NewModule
php artisan hex:add:migration NewModule
```

### 2. Adding API Endpoints

```bash
# Create API Platform resource
php artisan hex:add:resource ModuleName

# Create processors for operations
php artisan hex:add:processor ModuleName

# Create providers for queries
php artisan hex:add:provider ModuleName
```

### 3. Database Operations

```bash
# Create migration
php artisan hex:add:migration ModuleName

# Create factory for testing
php artisan hex:add:factory ModuleName

# Create seeder for sample data
php artisan hex:add:seeder ModuleName

# Run migrations
php artisan migrate

# Seed database
php artisan db:seed
```

### 4. Queue Jobs

```bash
# Create job
php artisan make:job ProcessDonationJob

# Process jobs
php artisan queue:work redis --queue=high,default,low

# Monitor queues
php artisan queue:monitor redis
```

### 5. Search Operations

```bash
# Configure Meilisearch
php artisan meilisearch:configure

# Import search indexes
php artisan scout:import-async 'Modules\Campaign\Domain\Model\Campaign'

# Monitor indexing
php artisan scout:index-monitor 'Modules\Campaign\Domain\Model\Campaign'
```

## Debugging Tips

### 1. Local Debugging with XDebug

```bash
# Enable XDebug in Herd (PHP 8.4)
# Edit: /Users/[username]/Library/Application Support/Herd/config/php/84/php.ini

[XDebug]
zend_extension=/Applications/Herd.app/Contents/Resources/xdebug/xdebug-84-arm64.so
xdebug.mode=debug,coverage
xdebug.start_with_request=yes
xdebug.client_host=localhost
xdebug.client_port=9003

# Disable JIT to prevent conflicts
opcache.jit=off
opcache.jit_buffer_size=0
```

### 2. Docker Debugging

```bash
# Access container shell
docker-compose exec app sh

# View application logs
docker-compose logs -f app

# Database access
docker-compose exec mysql mysql -u acme -p

# Redis CLI
docker-compose exec redis redis-cli
```

### 3. Common Debug Commands

```bash
# Laravel debugging
php artisan tinker                  # Interactive shell
php artisan route:list              # List all routes
php artisan config:show             # Show configuration

# Queue debugging
php artisan queue:failed            # Failed jobs
php artisan queue:retry all         # Retry failed jobs
php artisan horizon:status          # Horizon status

# Cache debugging
php artisan cache:clear             # Clear application cache
php artisan config:clear            # Clear config cache
php artisan route:clear             # Clear route cache
```

### 4. Performance Debugging

```bash
# Enable query logging in .env
DB_LOG_QUERIES=true

# Use Laravel Debugbar (development only)
composer require barryvdh/laravel-debugbar --dev

# Profile with Telescope (if installed)
php artisan telescope:install
```

### 5. API Debugging

```bash
# Test API endpoints
curl -X GET "http://localhost:8000/api/campaigns" \
     -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     -H "Accept: application/json"

# View API documentation
# Visit: http://localhost:8000/api (Interactive Swagger UI)

# Check API Platform resources
php artisan api:debug
```

## Development Workflows

### 1. Feature Development Workflow

```bash
# 1. Create feature branch
git checkout -b feature/new-donation-flow

# 2. Create module structure
php artisan hex:menu
# Select appropriate options

# 3. Write domain tests first (TDD)
./vendor/bin/pest tests/Unit/Donation/Domain/

# 4. Implement domain logic
# Edit domain models, value objects, etc.

# 5. Write application tests
./vendor/bin/pest tests/Unit/Donation/Application/

# 6. Implement application layer
# Create commands, queries, handlers

# 7. Write integration tests
./vendor/bin/pest tests/Integration/Donation/

# 8. Implement infrastructure
# Repositories, controllers, API resources

# 9. Write feature tests
./vendor/bin/pest tests/Feature/Api/Donation/

# 10. Run all quality checks
./vendor/bin/grumphp run

# 11. Commit and push
git add .
git commit -m "feat: add donation processing flow"
git push origin feature/new-donation-flow
```

### 2. Bug Fix Workflow

```bash
# 1. Create bugfix branch
git checkout -b bugfix/campaign-goal-calculation

# 2. Write failing test that reproduces bug
./vendor/bin/pest tests/Unit/Campaign/Domain/Model/CampaignTest.php::test_goal_calculation

# 3. Fix the bug in domain logic
# Edit the domain model

# 4. Verify test passes
./vendor/bin/pest tests/Unit/Campaign/Domain/Model/CampaignTest.php::test_goal_calculation

# 5. Run full test suite
./vendor/bin/pest

# 6. Run quality checks
./vendor/bin/grumphp run

# 7. Commit and push
git add .
git commit -m "fix: correct campaign goal calculation logic"
git push origin bugfix/campaign-goal-calculation
```

### 3. Daily Development Routine

```bash
# Morning routine
git pull origin dev                 # Get latest changes
composer install                   # Update dependencies
npm install                        # Update frontend deps
./vendor/bin/pest                  # Run test suite
php artisan migrate                # Run new migrations

# Development cycle
./vendor/bin/pest --filter="feature" # Run specific tests
./vendor/bin/pint                  # Fix code style
./vendor/bin/phpstan analyse       # Check static analysis

# End of day
./vendor/bin/grumphp run           # Full quality check
./vendor/bin/pest                  # Full test suite
git status                         # Check changes
```

## Key Documentation Links

### Architecture & Design
- [Module Structure Guide](../architecture/module-structure.md) - Detailed module organization
- [Hexagonal Architecture](../architecture/hexagonal-architecture.md) - Architecture principles
- [Module Interactions](../architecture/module-interactions.md) - How modules communicate
- [API Platform Design](../architecture/api-platform-design.md) - API architecture

### Development Guides
- [Getting Started](../development/getting-started.md) - Quick setup guide
- [Code Quality](../development/code-quality.md) - Quality standards
- [Hex Commands](../development/hex-commands.md) - Module generation commands
- [CI/CD Pipeline](../development/ci-cd-pipeline.md) - Deployment process

### Infrastructure & Operations
- [Docker Deployment](../docker/docker-deployment.md) - Docker setup
- [Meilisearch Troubleshooting](../infrastructure/meilisearch-troubleshooting.md) - Search issues
- [Queue Worker System](../infrastructure/queue-worker-system.md) - Background jobs
- [Redis Cache Usage](../infrastructure/redis-cache-usage.md) - Caching strategy

## FAQ

### Architecture Questions

**Q: Why Hexagonal Architecture over traditional Laravel MVC?**
A: Hexagonal Architecture provides better testability, maintainability, and team productivity. Business logic is isolated from framework concerns, making it easier to test and evolve.

**Q: How do I know which layer to put my code in?**
A: Follow this rule:
- **Domain**: Pure business logic, no framework dependencies
- **Application**: Orchestrates domain objects, handles use cases
- **Infrastructure**: Framework integration, external services, data persistence

**Q: When should I create a new module?**
A: Create a new module when you have a distinct business capability (e.g., Campaign, Donation, User, Notification). Each module should have its own domain models and be relatively independent.

### Testing Questions

**Q: Do I need to test everything?**
A: Focus on:
- Domain logic (high value, easy to test)
- Application handlers (medium value, medium complexity)
- API endpoints (medium value, higher complexity)
- Skip: Simple getters/setters (low value)
- Skip: Framework code (tested by framework)

**Q: How do I test external services?**
A: Use different strategies by layer:
- **Unit tests**: Mock external services
- **Integration tests**: Use real services or test doubles
- **Feature tests**: Mock external services to focus on API contract

### Development Questions

**Q: How do I add a new API endpoint?**
A:
1. Create domain model and business logic
2. Add command/query handlers in application layer
3. Create API Platform resource
4. Add processor (for writes) or provider (for reads)
5. Write tests for each layer

**Q: How do I handle database migrations?**
A: Use the hex commands:
```bash
php artisan hex:add:migration ModuleName
```
This creates migration in the correct module location.

**Q: How do I debug failing tests?**
A:
- Use `--filter` to run specific tests
- Add `dd()` or `dump()` for debugging
- Use `--stop-on-failure` to stop at first failure
- Check logs in `storage/logs/`

### Common Errors

**Q: "Class not found" error for domain models**
A: Make sure you've run `composer dump-autoload` after creating new classes.

**Q: PHPStan errors about missing properties**
A: Ensure all properties are declared and properly typed. Use `@var` annotations if needed.

**Q: Tests failing with database errors**
A: Check that test database is configured correctly in `.env.testing` and migrations have been run.

**Q: API Platform resource not showing up**
A: Ensure the resource class has proper `#[ApiResource]` attributes and is in the correct namespace.

## Getting Help

### Internal Resources
- **Documentation**: Check the `/docs` folder for specific guides
- **Code Reviews**: Submit PR for review
- **Technical Support**: Contact info@go2digit.al for technical assistance

### External Resources
- [Laravel Documentation](https://laravel.com/docs)
- [API Platform Documentation](https://api-platform.com/docs/)
- [Pest Testing Framework](https://pestphp.com/docs)
- [Hexagonal Architecture Guide](https://alistair.cockburn.us/hexagonal-architecture/)
- [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)

### Support Contacts
- **Technical Support**: info@go2digit.al
- **General Inquiries**: info@go2digit.al

---

## Welcome to the Team!

You're now ready to start contributing to the ACME Corp CSR Platform. Remember:

1. **Start Small**: Begin with simple features to understand the architecture
2. **Ask Questions**: Contact info@go2digit.al for technical support
3. **Follow TDD**: Write tests first, then implement
4. **Maintain Quality**: Run quality checks before committing
5. **Document**: Update documentation when you learn something new

**Happy Coding!**

---

*This guide is maintained by the Go2digit.al development team. Last updated: 2025-09-18*