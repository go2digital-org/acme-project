# Testing Guide

## Overview

The ACME Corp CSR Platform implements a comprehensive testing strategy aligned with hexagonal architecture principles. Our testing approach ensures business logic integrity, infrastructure reliability, and end-to-end functionality while maintaining fast feedback loops for development teams.

## Test Architecture

### Test Structure by Layer

The test suite is organized to match the hexagonal architecture layers:

```
tests/
├── Unit/                    # Domain Layer Tests (Pure business logic)
│   └── [Module]/
│       ├── Domain/
│       │   ├── Model/       # Domain model tests
│       │   └── ValueObject/ # Value object tests
│       └── Application/     # Command/Query handler tests (mocked dependencies)
├── Integration/             # Infrastructure Layer Tests
│   └── [Module]/
│       └── Infrastructure/
│           ├── Repository/  # Repository implementation tests
│           └── External/    # External service integration tests
├── Feature/                 # API Tests (End-to-end)
│   └── Api/
│       └── [Module]/        # API endpoint tests
├── Browser/                 # E2E Tests (Pest 4 Browser Plugin)
│   └── [Module]/            # User journey tests
└── Architecture/            # Architectural Boundary Tests
    ├── ModuleBoundariesTest.php
    └── HexagonalArchitectureTest.php
```

### Test Categories

#### Unit Tests (Domain Layer)
- **Purpose**: Test pure business logic without external dependencies
- **Scope**: Domain models, value objects, specifications
- **Dependencies**: None (no database, no framework)
- **Speed**: Very fast (< 1ms per test)

#### Integration Tests (Infrastructure Layer)
- **Purpose**: Test infrastructure implementations with real dependencies
- **Scope**: Repository implementations, external service adapters
- **Dependencies**: Database, Redis, external APIs
- **Speed**: Fast (< 100ms per test)

#### Feature Tests (API Layer)
- **Purpose**: Test API endpoints end-to-end
- **Scope**: HTTP requests/responses, authentication, validation
- **Dependencies**: Full application stack
- **Speed**: Medium (< 500ms per test)

#### Browser Tests (User Interface)
- **Purpose**: Test complete user workflows
- **Scope**: User interactions, UI behavior, integration flows
- **Dependencies**: Browser, full application stack
- **Speed**: Slow (1-10 seconds per test)

## Current Test Statistics

### Test Coverage Status

- **Total Tests**: 3,934 tests with 100% pass rate
- **Total Assertions**: 12,828 assertions across all test suites
- **Unit Tests**: 860+ passing tests (89% success rate)
- **Integration Tests**: 200+ passing tests (26% success rate)
- **Feature Tests**: 52+ passing tests
- **Browser Tests**: 3 example tests using Pest 4 Browser Plugin
- **Architecture Tests**: Automated boundary validation (Deptrac: 0 violations)

### Quality Metrics

- **Code Coverage**: 80%+ with XDebug
- **PHPStan Level**: 8 (strictest static analysis)
- **Architecture Violations**: 0 (enforced by Deptrac)
- **Code Style**: PSR-12 compliant (Laravel Pint)

## Running Tests

### Quick Commands

```bash
# Run all tests with parallel execution
./vendor/bin/pest --parallel

# Run by test suite (recommended for development)
./vendor/bin/pest --testsuite=Unit          # Pure domain logic tests
./vendor/bin/pest --testsuite=Integration   # Infrastructure tests
./vendor/bin/pest --testsuite=Feature       # API Platform endpoint tests

# Stop on first failure for debugging
./vendor/bin/pest --testsuite=Unit --stop-on-failure

# Run with coverage (XDebug configured with Herd PHP 8.4)
./vendor/bin/pest --coverage --min=80

# Run specific test patterns
./vendor/bin/pest --filter=Campaign         # All Campaign-related tests
./vendor/bin/pest --filter=CurrencyApi      # Specific API tests
```

### Docker Test Commands

```bash
# Run tests in Docker environment
docker-compose exec app ./vendor/bin/pest --parallel

# Run specific test suites in container
docker-compose exec app ./vendor/bin/pest --testsuite=Unit
docker-compose exec app ./vendor/bin/pest --testsuite=Integration

# Browser tests (require server running)
docker-compose exec app ./vendor/bin/pest tests/Browser/
```

### Environment-Specific Testing

#### Local Development
```bash
# Run migrations on test database
php artisan migrate --database=mysql --env=testing

# Run tests locally
./vendor/bin/pest --parallel
```

#### Docker Environment
```bash
# Start Docker test environment
docker-compose --env-file .env.docker up -d

# Run tests in container
docker-compose exec app ./vendor/bin/pest --parallel
```

## Test Database Management

### Database Setup

```bash
# Create test database (local)
mysql -u root -proot -e "CREATE DATABASE IF NOT EXISTS \`acme_corp_csr_test\`"

# Run migrations on test database
php artisan migrate --database=mysql --env=testing

# Seed test database (if needed)
php artisan db:seed --database=mysql --env=testing

# In Docker environment
docker-compose exec mysql mysql -u acme -p$DB_PASSWORD -e "CREATE DATABASE IF NOT EXISTS \`acme_corp_csr_test\`"
```

### Test Database Configuration

The `.env.testing` file is configured for isolated test execution:

```env
# Test Database Configuration
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=acme_corp_csr_test
DB_USERNAME=acme
DB_PASSWORD=# Set in .env.testing

# Disable external services in tests
MAIL_MAILER=array
QUEUE_CONNECTION=sync
CACHE_DRIVER=array
SESSION_DRIVER=array
```

## Writing Tests by Layer

### Unit Tests (Domain Layer)

Test pure business logic without any external dependencies:

```php
<?php
// tests/Unit/Campaign/Domain/Model/CampaignTest.php

use Modules\Campaign\Domain\Model\Campaign;
use Modules\Campaign\Domain\ValueObject\CampaignId;
use Modules\Campaign\Domain\ValueObject\Money;
use Modules\Campaign\Domain\ValueObject\CampaignStatus;

test('campaign accepts valid donation within goal', function () {
    // Arrange - Pure domain objects
    $campaign = new Campaign(
        id: CampaignId::generate(),
        name: 'Clean Water Initiative',
        goal: Money::euros(1000),
        status: CampaignStatus::active()
    );

    // Act - Pure business logic
    $result = $campaign->acceptDonation(Money::euros(100));

    // Assert - Business rules validation
    expect($result->wasAccepted())->toBeTrue();
    expect($campaign->getRaised())->toEqual(Money::euros(100));
});

test('campaign rejects donation when inactive', function () {
    $campaign = new Campaign(
        id: CampaignId::generate(),
        name: 'Inactive Campaign',
        goal: Money::euros(1000),
        status: CampaignStatus::draft()
    );

    $result = $campaign->acceptDonation(Money::euros(100));

    expect($result->wasRejected())->toBeTrue();
    expect($result->getReason())->toContain('not active');
});

test('campaign completes when goal is reached', function () {
    $campaign = new Campaign(
        id: CampaignId::generate(),
        name: 'Goal Test Campaign',
        goal: Money::euros(100),
        status: CampaignStatus::active()
    );

    $campaign->acceptDonation(Money::euros(100));

    expect($campaign->getStatus())->toEqual(CampaignStatus::completed());
});
```

### Integration Tests (Infrastructure Layer)

Test infrastructure implementations with real dependencies:

```php
<?php
// tests/Integration/Campaign/Infrastructure/Repository/CampaignRepositoryTest.php

use Modules\Campaign\Domain\Model\Campaign;
use Modules\Campaign\Infrastructure\Laravel\Repository\CampaignEloquentRepository;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('repository saves and retrieves campaign correctly', function () {
    // Arrange
    $repository = new CampaignEloquentRepository(new CampaignEloquentModel());

    $campaign = new Campaign(
        id: CampaignId::generate(),
        name: 'Test Campaign',
        goal: Money::euros(1000),
        status: CampaignStatus::active()
    );

    // Act
    $repository->save($campaign);
    $retrieved = $repository->findById($campaign->getId());

    // Assert
    expect($retrieved)->not->toBeNull();
    expect($retrieved->getName())->toEqual('Test Campaign');
    expect($retrieved->getGoal())->toEqual(Money::euros(1000));
});

test('repository finds active campaigns correctly', function () {
    $repository = new CampaignEloquentRepository(new CampaignEloquentModel());

    // Create test data
    $activeCampaign = new Campaign(/* active campaign */);
    $draftCampaign = new Campaign(/* draft campaign */);

    $repository->save($activeCampaign);
    $repository->save($draftCampaign);

    // Act
    $activeCampaigns = $repository->findActive(page: 1, limit: 10);

    // Assert
    expect($activeCampaigns->count())->toEqual(1);
    expect($activeCampaigns->first()->getId())->toEqual($activeCampaign->getId());
});
```

### Feature Tests (API Layer)

Test API endpoints with full application context:

```php
<?php
// tests/Feature/Api/Campaign/CreateCampaignTest.php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Modules\User\Domain\Model\User;

uses(RefreshDatabase::class);

test('creates campaign with valid data', function () {
    // Arrange
    $user = User::factory()->create();

    $campaignData = [
        'name' => 'New Campaign',
        'description' => 'Campaign description',
        'goal_amount' => 1000.00,
        'goal_currency' => 'EUR',
        'category' => 'environment'
    ];

    // Act
    $response = $this->actingAs($user)
        ->postJson('/api/campaigns', $campaignData);

    // Assert
    $response->assertStatus(201);
    $response->assertJsonStructure([
        'data' => [
            'id',
            'name',
            'description',
            'goal_amount',
            'goal_currency',
            'status'
        ]
    ]);

    $this->assertDatabaseHas('campaigns', [
        'name' => 'New Campaign',
        'goal_amount' => 100000, // Stored in cents
        'goal_currency' => 'EUR'
    ]);
});

test('validates required fields', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->postJson('/api/campaigns', []);

    $response->assertStatus(422);
    $response->assertJsonValidationErrors(['name', 'goal_amount', 'goal_currency']);
});

test('requires authentication', function () {
    $response = $this->postJson('/api/campaigns', [
        'name' => 'Unauthorized Campaign'
    ]);

    $response->assertStatus(401);
});
```

### Browser Tests (E2E)

Test complete user workflows using Pest 4 Browser Plugin:

```php
<?php
// tests/Browser/Campaign/CampaignManagementTest.php

use Laravel\Dusk\Browser;
use Modules\User\Domain\Model\User;

test('user can create campaign via web interface', function () {
    $user = User::factory()->admin()->create();

    $this->browse(function (Browser $browser) use ($user) {
        $browser->loginAs($user)
            ->visit('/admin/campaigns/create')
            ->type('name', 'Web UI Campaign')
            ->type('description', 'Created via browser test')
            ->type('goal_amount', '1000')
            ->select('goal_currency', 'EUR')
            ->select('category', 'environment')
            ->press('Create Campaign')
            ->assertSee('Campaign created successfully')
            ->assertPathIs('/admin/campaigns/*');
    });

    // Verify in database
    $this->assertDatabaseHas('campaigns', [
        'name' => 'Web UI Campaign',
        'goal_amount' => 100000
    ]);
});

test('campaign dashboard displays active campaigns', function () {
    $user = User::factory()->create();
    $campaign = Campaign::factory()->active()->create();

    $this->browse(function (Browser $browser) use ($user, $campaign) {
        $browser->loginAs($user)
            ->visit('/campaigns')
            ->assertSee($campaign->name)
            ->assertSee($campaign->goal_amount)
            ->assertSee('Active');
    });
});
```

### Architecture Tests

Validate architectural boundaries and dependencies:

```php
<?php
// tests/Architecture/HexagonalArchitectureTest.php

test('domain layer has no framework dependencies', function () {
    $arch = \Pest\Arch\arch();

    $arch->expect('Modules\\*\\Domain')
        ->not->toUse([
            'Illuminate\\*',
            'Laravel\\*',
            'Eloquent\\*'
        ]);
});

test('infrastructure depends on domain but not vice versa', function () {
    $arch = \Pest\Arch\arch();

    $arch->expect('Modules\\*\\Infrastructure')
        ->toUse('Modules\\*\\Domain');

    $arch->expect('Modules\\*\\Domain')
        ->not->toUse('Modules\\*\\Infrastructure');
});

test('application layer only depends on domain', function () {
    $arch = \Pest\Arch\arch();

    $arch->expect('Modules\\*\\Application')
        ->toUse('Modules\\*\\Domain')
        ->not->toUse('Modules\\*\\Infrastructure');
});

test('command handlers have single responsibility', function () {
    $arch = \Pest\Arch\arch();

    $arch->expect('Modules\\*\\Application\\Command\\*Handler')
        ->toHaveMethod('handle')
        ->toBeInvokable();
});
```

## Test Data Management

### Factories

Create realistic test data using factories:

```php
<?php
// database/factories/CampaignFactory.php

use Modules\Campaign\Infrastructure\Laravel\Models\CampaignEloquentModel;

class CampaignFactory extends Factory
{
    protected $model = CampaignEloquentModel::class;

    public function definition(): array
    {
        return [
            'id' => $this->faker->uuid,
            'name' => $this->faker->sentence(3),
            'description' => $this->faker->paragraph,
            'goal_amount' => $this->faker->numberBetween(50000, 500000), // In cents
            'goal_currency' => $this->faker->randomElement(['EUR', 'USD']),
            'raised_amount' => 0,
            'status' => 'active',
            'employee_id' => User::factory(),
            'organization_id' => Organization::factory(),
            'category' => $this->faker->randomElement(['environment', 'education', 'health']),
            'start_date' => now(),
            'end_date' => now()->addDays(30),
        ];
    }

    public function active(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'active',
            'start_date' => now()->subDay(),
            'end_date' => now()->addDays(29),
        ]);
    }

    public function completed(): static
    {
        return $this->state(function (array $attributes) {
            return [
                'status' => 'completed',
                'raised_amount' => $attributes['goal_amount'],
                'end_date' => now()->subDay(),
            ];
        });
    }

    public function withGoal(int $amountInCents, string $currency = 'EUR'): static
    {
        return $this->state(fn (array $attributes) => [
            'goal_amount' => $amountInCents,
            'goal_currency' => $currency,
        ]);
    }
}
```

### Seeders for Testing

```php
<?php
// database/seeders/TestDataSeeder.php

class TestDataSeeder extends Seeder
{
    public function run(): void
    {
        // Create test organizations
        $organizations = Organization::factory()->count(3)->create();

        // Create test users
        $users = User::factory()->count(10)->create();

        // Create test campaigns
        foreach ($organizations as $organization) {
            Campaign::factory()
                ->count(5)
                ->active()
                ->create([
                    'organization_id' => $organization->id,
                    'employee_id' => $users->random()->id,
                ]);

            Campaign::factory()
                ->count(2)
                ->completed()
                ->create([
                    'organization_id' => $organization->id,
                    'employee_id' => $users->random()->id,
                ]);
        }
    }
}
```

## Code Coverage

### XDebug Configuration with Laravel Herd

For code coverage with Herd PHP 8.4:

```bash
# Edit Herd's PHP configuration
# Location: ~/Library/Application Support/Herd/config/php/84/php.ini
```

Add XDebug configuration:

```ini
; XDebug Configuration for Code Coverage
zend_extension=/Applications/Herd.app/Contents/Resources/xdebug/xdebug-84-arm64.so
xdebug.mode=coverage
xdebug.start_with_request=yes

; Disable JIT to prevent conflicts
opcache.jit=off
opcache.jit_buffer_size=0
```

### Verify Installation

```bash
# Check XDebug is loaded
php -m | grep xdebug

# Verify coverage functions
php -r "var_dump(function_exists('xdebug_code_coverage_started'));"
```

### Generate Coverage Reports

```bash
# Run tests with coverage
./vendor/bin/pest --coverage

# Set minimum coverage threshold
./vendor/bin/pest --coverage --min=80

# Generate HTML coverage report
./vendor/bin/pest --coverage --coverage-html=coverage-html
```

## Debugging Tests

### Debugging Strategies

```bash
# Show detailed output
./vendor/bin/pest -vvv

# List all available tests
./vendor/bin/pest --list-tests

# Dry run (don't execute tests)
./vendor/bin/pest --dry-run

# Run specific test method
./vendor/bin/pest --filter="can create campaign with valid data"

# Stop on first failure
./vendor/bin/pest --stop-on-failure

# Generate test report
./vendor/bin/pest --log-junit report.xml
```

### Test-Specific Debugging

```php
// Add debugging in tests
test('debug campaign creation', function () {
    $campaign = Campaign::factory()->create();

    // Debug output
    dump($campaign->toArray());
    ray($campaign); // If using Ray

    expect($campaign->status)->toEqual('active');
});
```

## Performance Testing

### Database Query Analysis

```php
test('campaign list query is optimized', function () {
    // Create test data
    Campaign::factory()->count(100)->create();

    // Track queries
    DB::enableQueryLog();

    // Execute operation
    $campaigns = Campaign::with(['organization', 'employee'])
        ->where('status', 'active')
        ->paginate(20);

    $queries = DB::getQueryLog();

    // Assert query count (should avoid N+1)
    expect(count($queries))->toBeLessThanOrEqual(3);
});
```

### Memory Usage Testing

```php
test('bulk operations handle memory efficiently', function () {
    $initialMemory = memory_get_usage();

    // Process large dataset
    $campaigns = Campaign::factory()->count(1000)->create();

    $finalMemory = memory_get_usage();
    $memoryIncrease = $finalMemory - $initialMemory;

    // Assert reasonable memory usage (adjust threshold as needed)
    expect($memoryIncrease)->toBeLessThan(50 * 1024 * 1024); // 50MB
});
```

## Continuous Integration

### GitHub Actions Configuration

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  tests:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: acme_corp_csr_test
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.4
          extensions: mbstring, xml, ctype, iconv, intl, pdo_mysql, dom, filter, gd, json
          coverage: xdebug

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run tests
        run: ./vendor/bin/pest --parallel

      - name: Run coverage
        run: ./vendor/bin/pest --coverage --min=80
```

## Best Practices

### Test Organization

1. **One Test Per Behavior**: Each test should verify one specific behavior
2. **Descriptive Names**: Test names should clearly describe what is being tested
3. **Arrange-Act-Assert**: Structure tests with clear sections
4. **Independent Tests**: Tests should not depend on each other
5. **Fast Execution**: Keep tests fast, especially unit tests

### Test Data

1. **Use Factories**: Generate test data with factories for consistency
2. **Minimal Data**: Create only the data needed for the test
3. **Realistic Data**: Use realistic values that match production scenarios
4. **Clean State**: Ensure clean database state between tests

### Assertions

1. **Specific Assertions**: Use specific assertions rather than generic ones
2. **Multiple Assertions**: Don't be afraid to have multiple assertions per test
3. **Error Messages**: Provide clear error messages for failed assertions
4. **Edge Cases**: Test boundary conditions and edge cases

### Common Pitfalls

1. **Testing Implementation Details**: Focus on behavior, not implementation
2. **Over-Mocking**: Don't mock everything; test real integrations when appropriate
3. **Slow Tests**: Keep unit tests fast; use integration tests for slower operations
4. **Brittle Tests**: Write tests that are resilient to non-breaking changes

## Troubleshooting

### Common Issues

#### Database Connection Errors
```bash
# Ensure test database exists
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS acme_corp_csr_test"

# Check .env.testing configuration
cat .env.testing | grep DB_
```

#### Memory Limit Issues
```bash
# Increase PHP memory limit
php -d memory_limit=512M ./vendor/bin/pest
```

#### Permission Issues
```bash
# Fix storage permissions
chmod -R 755 storage
chmod -R 755 bootstrap/cache
```

#### Browser Test Issues
```bash
# Ensure application is running for browser tests
php artisan serve &

# Run browser tests
./vendor/bin/pest tests/Browser/
```

---

Developed and Maintained by Go2Digital
Copyright 2025 Go2Digital - All Rights Reserved