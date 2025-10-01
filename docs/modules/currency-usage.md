# Currency Helper Usage Examples

This document provides examples of how to use the new currency formatting functionality.

## CurrencyHelper Class Usage

```php
use Modules\Shared\Application\Helper\CurrencyHelper;

// Format Euro (European formatting: €1.234,56)
echo CurrencyHelper::formatEuro(1234.56); // Output: €1.234,56

// Format different currencies
echo CurrencyHelper::formatCurrency(1234.56, 'EUR'); // Output: €1.234,56
echo CurrencyHelper::formatCurrency(1234.56, 'USD'); // Output: $1,234.56
echo CurrencyHelper::formatCurrency(1234.56, 'GBP'); // Output: £1,234.56

// Parse currency strings back to floats
$amount = CurrencyHelper::parseCurrency('€1.234,56', 'EUR'); // Returns: 1234.56

// Get currency symbol
echo CurrencyHelper::getCurrencySymbol('EUR'); // Output: €

// Get supported currencies
$currencies = CurrencyHelper::getSupportedCurrencies(); // ['EUR', 'USD', 'GBP']
```

## Money Value Object Usage

```php
use Modules\Campaign\Domain\ValueObject\Money;

// Create Money with EUR as default currency
$money = new Money(1234.56); // Default currency is now EUR
echo $money->currency; // Output: EUR

// Format as Euro (European style)
echo $money->formatEuro(); // Output: €1.234,56

// Standard format
echo $money->format(); // Output: 1,234.56 EUR

// Create with specific currency
$usdMoney = new Money(1234.56, 'USD');
echo $usdMoney->format(); // Output: 1,234.56 USD
```

## Blade Directive Usage

**Note**: Some directives shown in examples below are not fully implemented yet.

### Currently Implemented Directives

In your Blade templates, you can use these directives:

```blade
{{-- Format with current user's currency --}}
@formatCurrency($campaign->goal_amount)
{{-- Output: €1.234,56 (based on session currency) --}}

{{-- Get current currency symbol --}}
@currencySymbol
{{-- Output: € --}}

{{-- Get current currency code --}}
@currencyCode
{{-- Output: EUR --}}
```

### Planned Directives (Not Yet Implemented)

```blade
{{-- These directives are planned but not yet available --}}
@euro($campaign->goal_amount)           {{-- Not implemented --}}
@currency($donation->amount, 'USD')     {{-- Not implemented --}}
@parseCurrency('€1.234,56', 'EUR')     {{-- Not implemented --}}
```

## Example in a Campaign View

```blade
<div class="campaign-card">
    <h3>{{ $campaign->title }}</h3>
    <div class="progress">
        <span class="current">@formatCurrency($campaign->current_amount)</span>
        <span class="target">of @formatCurrency($campaign->goal_amount)</span>
    </div>
    <div class="percentage">
        {{ number_format($campaign->getProgressPercentage(), 1) }}% funded
    </div>
</div>
```

## Benefits

1. **Consistent Formatting**: All currency formatting follows European standards for EUR
2. **Hexagonal Architecture**: Helper is in Application layer, following clean architecture
3. **Type Safety**: Value objects maintain currency validation
4. **Blade Integration**: Easy to use in templates with custom directives
5. **Extensible**: Easy to add new currencies or formatting rules

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved