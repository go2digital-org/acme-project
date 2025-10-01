# ADR-005: Employee to User Terminology Migration

## Status

Accepted

## Context

The ACME Corp CSR Platform was initially designed with the concept of "Employee" as the primary user entity, reflecting the internal corporate nature of the platform. However, as the platform evolved, several issues emerged:

### Problems with "Employee" Terminology:
1. **Scope Limitation**: Platform now supports external stakeholders (contractors, partners, vendors)
2. **Semantic Confusion**: "Employee" implies employment relationship, which doesn't apply to all users
3. **API Inconsistency**: External APIs and integrations expect "User" terminology
4. **Third-party Integration**: External services (OAuth providers, payment processors) use "User"
5. **Scalability**: Future plans include public campaigns accessible to non-employees

### Current System State:
- Database tables use `employees` (legacy)
- Domain models use `Employee` class names
- API endpoints use `/employees/*` routes
- Authentication system references "employee" throughout
- Frontend components reference "employee" concepts

### Stakeholder Concerns:
- **Business**: Wants platform to support broader user base
- **Development**: Consistency across codebase and external integrations
- **Marketing**: Public-facing features require user-friendly terminology
- **Legal**: Compliance requirements for external user data handling

## Decision

We will migrate from **"Employee"** to **"User"** terminology throughout the entire platform while maintaining backward compatibility during the transition period.

### Migration Strategy:
1. **Phase 1**: Domain layer migration (new User classes alongside Employee)
2. **Phase 2**: API layer migration (new endpoints with deprecation notices)
3. **Phase 3**: Database migration (table rename with migration scripts)
4. **Phase 4**: Frontend migration (component and terminology updates)
5. **Phase 5**: Legacy cleanup (remove deprecated Employee references)

### Implementation Approach:
```
Old Structure:              New Structure:
Employee                 -> User
EmployeeProfile         -> UserProfile
EmployeeRepository      -> UserRepository
EmployeeService         -> UserService
employees table         -> users table
/api/employees/*        -> /api/users/*
```

## Consequences

### Positive:
- **Broader Scope**: Platform can support non-employee users
- **Industry Standards**: Aligns with common terminology across platforms
- **Integration Friendly**: Easier integration with third-party services
- **Future-Proof**: Supports planned public campaign features
- **Consistency**: Unified terminology across all platform layers
- **Marketing**: More appealing for public-facing features

### Negative:
- **Migration Complexity**: Large codebase changes required
- **Temporary Inconsistency**: Mixed terminology during transition
- **Breaking Changes**: API changes affect existing integrations
- **Data Migration**: Database schema changes require careful planning
- **Documentation Updates**: All documentation needs revision

### Risk Mitigation:
- **Gradual Migration**: Phase-by-phase approach to minimize disruption
- **Backward Compatibility**: Maintain old APIs during transition period
- **Comprehensive Testing**: Extensive test coverage for migration scripts
- **Clear Communication**: Deprecation notices and migration guides
- **Rollback Plan**: Ability to revert changes if issues arise

## Migration Plan

### Phase 1: Domain Layer (Week 1-2)
```php
// New User domain model
final class User
{
    private UserId $id;
    private string $email;
    private string $firstName;
    private string $lastName;
    private UserStatus $status;
    private UserProfile $profile;

    // Business logic methods
}

// Maintain Employee as alias during transition
class Employee extends User
{
    // Temporary compatibility layer
    public function getEmployeeId(): EmployeeId
    {
        return EmployeeId::fromString($this->id->toString());
    }
}
```

### Phase 2: Application Layer (Week 3-4)
```php
// New command handlers
final class CreateUserHandler
{
    public function handle(CreateUserCommand $command): User
    {
        // Implementation using new User model
    }
}

// Legacy handler delegates to new one
final class CreateEmployeeHandler
{
    public function __construct(private CreateUserHandler $userHandler) {}

    public function handle(CreateEmployeeCommand $command): Employee
    {
        $userCommand = new CreateUserCommand(
            $command->email,
            $command->firstName,
            $command->lastName
        );

        $user = $this->userHandler->handle($userCommand);
        return Employee::fromUser($user); // Compatibility wrapper
    }
}
```

### Phase 3: API Layer (Week 5-6)
```php
// New API routes
Route::prefix('api/v2')->group(function () {
    Route::apiResource('users', UserController::class);
    Route::post('users/{user}/activate', [UserController::class, 'activate']);
});

// Legacy routes with deprecation
Route::prefix('api/v1')->group(function () {
    Route::apiResource('employees', EmployeeController::class)
        ->deprecated('2024-01-01', 'Use /api/v2/users instead');
});

// API Platform resources
#[ApiResource(
    uriTemplate: '/users',
    operations: [new GetCollection(), new Post()],
    deprecationReason: 'Use /api/v2/users instead'
)]
class Employee extends User
{
    // Compatibility wrapper
}
```

### Phase 4: Database Migration (Week 7-8)
```sql
-- Step 1: Create new users table
CREATE TABLE users (
    id CHAR(36) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    status ENUM('active', 'inactive', 'suspended') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Step 2: Migrate data
INSERT INTO users (id, email, first_name, last_name, status, created_at, updated_at)
SELECT id, email, first_name, last_name, status, created_at, updated_at
FROM employees;

-- Step 3: Update foreign key references
ALTER TABLE campaigns ADD COLUMN user_id CHAR(36);
UPDATE campaigns c
SET user_id = (SELECT id FROM users WHERE id = c.employee_id);

-- Step 4: Create view for backward compatibility
CREATE VIEW employees AS
SELECT id, email, first_name, last_name, status, created_at, updated_at
FROM users;
```

### Phase 5: Frontend Migration (Week 9-10)
```typescript
// New user types
interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  status: UserStatus;
  profile: UserProfile;
}

// Legacy type alias
type Employee = User;

// Component updates
const UserProfile: React.FC<{ user: User }> = ({ user }) => {
  return (
    <div>
      <h1>{user.firstName} {user.lastName}</h1>
      <p>{user.email}</p>
    </div>
  );
};

// Legacy component wrapper
const EmployeeProfile: React.FC<{ employee: Employee }> = ({ employee }) => {
  return <UserProfile user={employee} />;
};
```

## Testing Strategy

### Migration Tests:
```php
// Test data consistency during migration
public function testEmployeeToUserDataMigration(): void
{
    $employee = Employee::factory()->create();

    $this->artisan('migrate:employee-to-user');

    $user = User::where('email', $employee->email)->first();
    $this->assertNotNull($user);
    $this->assertEquals($employee->firstName, $user->firstName);
    $this->assertEquals($employee->lastName, $user->lastName);
}

// Test backward compatibility
public function testEmployeeApiBackwardCompatibility(): void
{
    $user = User::factory()->create();

    $response = $this->getJson('/api/v1/employees/' . $user->id);

    $response->assertOk();
    $response->assertJsonStructure([
        'id', 'email', 'first_name', 'last_name', 'status'
    ]);
}
```

## Rollback Strategy

In case of critical issues during migration:

1. **Database Rollback**: Revert to employees table using backup
2. **Code Rollback**: Git revert to pre-migration commit
3. **API Rollback**: Disable new endpoints, re-enable old ones
4. **Cache Clear**: Clear all application and database caches

## Communication Plan

### Internal Teams:
- **Week -2**: Announce migration plan to all development teams
- **Week 0**: Provide migration timeline and breaking changes list
- **Week 4**: Mid-migration status update and API deprecation notices
- **Week 8**: Final migration completion and legacy cleanup timeline

### External Partners:
- **Week -1**: Email notification about upcoming API changes
- **Week 0**: Documentation updates with migration guide
- **Week 4**: Deprecation headers in API responses
- **Week 12**: Final notice before legacy API removal

## Related Decisions

- [ADR-004: Domain Driven Design](004-domain-driven-design.md) - Domain modeling principles
- [ADR-002: API Platform](002-api-platform.md) - API implementation approach
- [ADR-001: Hexagonal Architecture](001-hexagonal-architecture.md) - Overall architecture pattern

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved