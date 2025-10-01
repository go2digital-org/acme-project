# ACME Corp - Unified Linting Guide

## Overview

This project uses a comprehensive linting system orchestrated through Composer scripts and GrumPHP. All tools are optimized for speed, parallel execution, and developer productivity.

## Unified Linting Commands

### Quick Commands

```bash
# Standard linting with GrumPHP
composer lint

# Includes:
# - Rector checks and Pint auto-fix
# - PHPStan static analysis
# - Deptrac architecture validation
# - Security checks
```

### Code Style Fixes

```bash
# Laravel Pint - fix code style automatically
composer lint:fix
# OR
vendor/bin/pint
```

### Static Analysis

```bash
# PHPStan analysis
composer stan

# Generate PHPStan baseline
composer stan:baseline
```

### Architecture Validation

```bash
# Deptrac - validate module boundaries
composer deptrac
```

## Specialized Scripts

### Pre-commit Hooks

```bash
# Run pre-commit checks
composer pre-commit

# Includes:
# - Git blacklist check
# - Rector + Pint auto-fix
# - PHPStan analysis
```

### Code Refactoring

```bash
# Rector - safe refactoring checks
composer rector:check-safe

# Apply safe refactoring
composer rector

# Dry run mode
composer rector:dry

# Aggressive refactoring (dry run)
composer rector:aggressive

# Apply aggressive refactoring
composer rector:aggressive:apply
```

## Individual Tool Scripts

### Code Style

```bash
# Laravel Pint (Primary code style tool)
vendor/bin/pint            # Fix all style issues
vendor/bin/pint --test     # Check style without fixing

# PHP CS Fixer (Available as backup tool)
vendor/bin/php-cs-fixer fix               # Fix with PHP CS Fixer
vendor/bin/php-cs-fixer fix --dry-run     # Preview changes
```

### Static Analysis

```bash
# PHPStan (Comprehensive analysis)
composer stan              # Run analysis
composer stan:baseline     # Generate baseline
vendor/bin/phpstan analyse # Direct command

# Note: Psalm is configured but specific composer shortcuts not available
vendor/bin/psalm           # Run Psalm directly
```

### Architecture Validation

```bash
# Deptrac (Architecture boundaries)
composer deptrac           # Validate architecture rules
```

### Code Refactoring

```bash
# Rector (Available commands)
composer rector                    # Apply safe refactoring rules
composer rector:dry               # Preview safe changes
composer rector:check-safe        # Check safe refactoring (dry run)
composer rector:aggressive        # Preview aggressive changes
composer rector:aggressive:apply  # Apply aggressive refactoring

# Direct commands
vendor/bin/rector process --config=rector/rector-safe.php
vendor/bin/rector process --config=rector/rector.php --dry-run
```

### Security Scanning

```bash
# Security vulnerability scanning
vendor/bin/security-checker security:check  # Check for vulnerabilities

# Note: Security scanning is integrated into GrumPHP linting
# No separate composer security commands available
```

## Performance Characteristics

### Speed Optimization Features

1. **Parallel Execution**: Up to 8 workers for maximum speed
2. **Intelligent Caching**: All tools use caching when possible
3. **Incremental Analysis**: Only analyze changed files when supported
4. **Priority-Based Ordering**: Fastest tools run first
5. **Early Termination**: Optional stop-on-failure for rapid feedback

### Available Commands

| Command | Purpose |
|---------|----------|
| `composer lint` | Standard linting workflow |
| `composer lint:fix` | Auto-fix code style |
| `composer pre-commit` | Pre-commit hook validation |
| `composer stan` | PHPStan static analysis |
| `composer deptrac` | Architecture validation |

## Tool Integration

### GrumPHP Integration

GrumPHP test suites available:

- **quick**: Essential checks (git blacklist, rector + pint)
- **standard**: Balanced workflow (adds PHPStan, Deptrac)
- **full**: Comprehensive analysis (adds security, composer validation)
- **git_pre_commit**: Pre-commit hook optimized
- **git_commit_msg**: Commit message validation

### Laravel Integration

- **Pint Configuration**: Uses project-specific rules
- **PHPStan**: Laravel-optimized with Larastan
- **Psalm**: Laravel plugin enabled
- **Rector**: Laravel-specific refactoring rules

## Best Practices

### Development Workflow

1. **During Development**: Run `composer lint:fix` for style fixes
2. **Before Commit**: Run `composer pre-commit` or `composer lint`
3. **Static Analysis**: Run `composer stan` when needed
4. **Architecture Check**: Run `composer deptrac` for module boundaries

### Fixing Issues

1. **Auto-fix First**: Always run `composer lint:fix`
2. **Manual Review**: Check auto-fixes before committing
3. **Iterative Approach**: Fix issues in small batches
4. **Baseline Updates**: Update baselines when needed

### Git Hook Integration

```bash
# Install git hooks (if not already configured)
vendor/bin/grumphp git:init
```

## Configuration Files

### Primary Configurations

- `grumphp.yml` - Main orchestration configuration
- `phpstan.neon` - PHPStan static analysis rules
- `psalm.xml` - Psalm configuration and rules
- `deptrac.yaml` - Architecture boundary definitions
- `rector.php` - Code refactoring rules (plus profile variants)

### Performance Tuning

All configurations are optimized for:
- Maximum parallel execution
- Intelligent caching
- Minimal memory usage
- Fast feedback loops
- Developer productivity

## Troubleshooting

### Common Issues

1. **Memory Limits**: Increase PHP memory limit if needed
2. **Cache Issues**: Clear caches with `composer clear-cache`
3. **Baseline Issues**: Regenerate baselines when upgrading tools
4. **Performance**: Check that caching is enabled

### Getting Help

```bash
# View available scripts
composer list

# View GrumPHP configuration
vendor/bin/grumphp git:init --help

# Tool-specific help
vendor/bin/phpstan --help
vendor/bin/psalm --help
vendor/bin/deptrac --help
```

## Integration Examples

### IDE Integration

Most IDEs can be configured to run these commands:
- **PHPStorm**: Configure as external tools
- **VSCode**: Add to tasks.json
- **Vim/NeoVim**: Use appropriate plugins

### Git Workflow

```bash
# Complete pre-commit workflow
git add .
composer pre-commit    # Auto-fixes and validates
git add .             # Add any auto-fixes
git commit -m "feat: add new feature"
composer lint:full    # Final validation before push
git push
```

This unified system provides fast, comprehensive code quality assurance while maintaining developer productivity.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved