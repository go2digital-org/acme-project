# Internationalization Architecture

## Strategic Localization Decision

The implementation of comprehensive internationalization in the ACME Corp CSR platform demonstrates enterprise-level planning for global deployment. This document details the technical leadership decisions behind our multilingual architecture and implementation strategy.

## Architecture Overview

### URL-Based Localization Strategy

**Decision**: Implement URL-based localization with Laravel Localization package

**Rationale**:
- **SEO Optimization**: Search engines can index different language versions separately
- **User Experience**: Clear language indication in URLs improves user understanding
- **CDN Compatibility**: Geographic content delivery optimization
- **Enterprise Integration**: Corporate intranets can route by language/region
- **Analytics Clarity**: Clear segmentation of traffic by language

### URL Structure Implementation

```php
// routes/web.php
Route::group([
    'prefix' => LaravelLocalization::setLocale(),
    'middleware' => ['localeSessionRedirect', 'localizationRedirect', 'localeViewPath']
], function() {
    // All routes are automatically prefixed with locale
    Route::get('/', DashboardController::class)->name('dashboard');
    Route::get('/campaigns', [ListCampaignsController::class])->name('campaigns.index');
    Route::get('/campaigns/{campaign}', [ShowCampaignController::class])->name('campaigns.show');
    Route::get('/donate/{campaign}', [DonateController::class])->name('campaigns.donate');
});

// Results in URLs:
// /en/campaigns - English campaigns
// /fr/campaigns - French campaigns  
// /nl/campaigns - Dutch campaigns
```

## Language Detection & Persistence Strategy

### Multi-Layer Language Detection

```php
// app/Helpers/LocalizationHelper.php
declare(strict_types=1);

namespace App\\Helpers;

use Illuminate\\Support\\Facades\\Session;
use Mcamara\\LaravelLocalization\\Facades\\LaravelLocalization;

final class LocalizationHelper
{
    /**
     * Get current locale with fallback chain:
     * 1. URL parameter (highest priority)
     * 2. Session storage (user choice persistence)
     * 3. Cookie storage (cross-session persistence)  
     * 4. Browser Accept-Language header
     * 5. Application default (fallback)
     */
    public static function getCurrentLocale(): string
    {
        // URL takes absolute priority
        $urlLocale = LaravelLocalization::getCurrentLocale();
        if ($urlLocale && self::isLocaleSupported($urlLocale)) {
            return $urlLocale;
        }
        
        // Check user session preference
        $sessionLocale = Session::get('locale');
        if ($sessionLocale && self::isLocaleSupported($sessionLocale)) {
            return $sessionLocale;
        }
        
        // Check cookie preference  
        $cookieLocale = request()->cookie('laravel_localization');
        if ($cookieLocale && self::isLocaleSupported($cookieLocale)) {
            return $cookieLocale;
        }
        
        // Browser language detection
        $browserLocale = self::detectBrowserLocale();
        if ($browserLocale && self::isLocaleSupported($browserLocale)) {
            return $browserLocale;
        }
        
        // Application default
        return config('app.locale', 'en');
    }
    
    /**
     * Intelligent browser language detection
     */
    private static function detectBrowserLocale(): ?string
    {
        $acceptLanguage = request()->server('HTTP_ACCEPT_LANGUAGE');
        if (!$acceptLanguage) {
            return null;
        }
        
        $languages = [];
        preg_match_all('/([a-z]{1,8}(-[a-z]{1,8})?)(;q=([0-9.]+))?/i', 
            $acceptLanguage, $matches, PREG_SET_ORDER);
        
        foreach ($matches as $match) {
            $lang = strtolower($match[1]);
            $priority = $match[4] ?? '1.0';
            $languages[$lang] = (float) $priority;
        }
        
        arsort($languages);
        
        foreach (array_keys($languages) as $browserLang) {
            $locale = self::mapBrowserLocaleToSupported($browserLang);
            if ($locale) {
                return $locale;
            }
        }
        
        return null;
    }
    
    /**
     * Map browser locales to supported application locales
     */
    private static function mapBrowserLocaleToSupported(string $browserLocale): ?string
    {
        $mapping = [
            'en' => 'en',
            'en-us' => 'en',
            'en-gb' => 'en',
            'fr' => 'fr', 
            'fr-fr' => 'fr',
            'fr-ca' => 'fr',
            'nl' => 'nl',
            'nl-nl' => 'nl',
            'nl-be' => 'nl'
        ];
        
        return $mapping[$browserLocale] ?? null;
    }
    
    public static function isLocaleSupported(string $locale): bool
    {
        return in_array($locale, config('laravellocalization.supportedLocales', []), true);
    }
}
```

## Database Internationalization Strategy

### Model Translation Architecture

```php
// modules/Shared/Domain/Traits/HasTranslations.php
declare(strict_types=1);

namespace Modules\\Shared\\Domain\\Traits;

trait HasTranslations
{
    private array $translations = [];
    
    public function getTranslation(string $field, ?string $locale = null): ?string
    {
        $locale = $locale ?? app()->getLocale();
        
        return $this->translations[$locale][$field] ?? 
               $this->translations[config('app.fallback_locale')][$field] ?? 
               null;
    }
    
    public function setTranslation(string $field, string $locale, string $value): void
    {
        $this->translations[$locale][$field] = $value;
    }
    
    public function hasTranslation(string $field, string $locale): bool
    {
        return isset($this->translations[$locale][$field]);
    }
    
    public function getAllTranslations(): array
    {
        return $this->translations;
    }
}

// Usage in Domain Models
// modules/Campaign/Domain/Model/Campaign.php
declare(strict_types=1);

namespace Modules\\Campaign\\Domain\\Model;

use Modules\\Shared\\Domain\\Traits\\HasTranslations;

final class Campaign
{
    use HasTranslations;
    
    public function __construct(
        private readonly CampaignId $id,
        private array $names = [],
        private array $descriptions = []
    ) {
        $this->translations = [
            'names' => $names,
            'descriptions' => $descriptions
        ];
    }
    
    public function getName(?string $locale = null): string
    {
        return $this->getTranslation('names', $locale) ?? 'Untitled Campaign';
    }
    
    public function getDescription(?string $locale = null): string
    {
        return $this->getTranslation('descriptions', $locale) ?? '';
    }
    
    public function setName(string $name, string $locale): void
    {
        $this->setTranslation('names', $locale, $name);
    }
    
    public function setDescription(string $description, string $locale): void
    {
        $this->setTranslation('descriptions', $locale, $description);
    }
}
```

### Database Schema for Translations

```sql
-- Translation-ready schema design
CREATE TABLE campaigns (
    id VARCHAR(36) PRIMARY KEY,
    goal_amount DECIMAL(10,2) NOT NULL,
    goal_currency VARCHAR(3) NOT NULL,
    raised_amount DECIMAL(10,2) DEFAULT 0,
    status VARCHAR(20) NOT NULL,
    employee_id VARCHAR(36) NOT NULL,
    organization_id VARCHAR(36) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_campaigns_status (status),
    INDEX idx_campaigns_organization (organization_id)
);

CREATE TABLE campaign_translations (
    id VARCHAR(36) PRIMARY KEY,
    campaign_id VARCHAR(36) NOT NULL,
    locale VARCHAR(5) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (campaign_id) REFERENCES campaigns(id) ON DELETE CASCADE,
    UNIQUE KEY uk_campaign_locale (campaign_id, locale),
    INDEX idx_campaign_translations_locale (locale)
);
```

### Repository Implementation with Translation Support

```php
// modules/Campaign/Infrastructure/Laravel/Repository/CampaignEloquentRepository.php
declare(strict_types=1);

namespace Modules\\Campaign\\Infrastructure\\Laravel\\Repository;

use Modules\\Campaign\\Domain\\Model\\Campaign;
use Modules\\Campaign\\Domain\\Repository\\CampaignRepositoryInterface;
use Modules\\Campaign\\Infrastructure\\Laravel\\Models\\CampaignEloquentModel;

final readonly class CampaignEloquentRepository implements CampaignRepositoryInterface
{
    public function __construct(
        private CampaignEloquentModel $model
    ) {}
    
    public function findById(CampaignId $id): ?Campaign
    {
        $model = $this->model
            ->with('translations')
            ->find($id->toString());
            
        if (!$model) {
            return null;
        }
        
        return $this->toDomainModel($model);
    }
    
    public function save(Campaign $campaign): void
    {
        $model = $this->model->firstOrNew(['id' => $campaign->getId()->toString()]);
        
        $model->fill([
            'id' => $campaign->getId()->toString(),
            'goal_amount' => $campaign->getGoal()->getAmount(),
            'goal_currency' => $campaign->getGoal()->getCurrency(),
            'raised_amount' => $campaign->getRaised()->getAmount(),
            'status' => $campaign->getStatus()->toString(),
            'employee_id' => $campaign->getEmployeeId()->toString(),
            'organization_id' => $campaign->getOrganizationId()->toString(),
        ]);
        
        $model->save();
        
        // Save translations
        $this->saveTranslations($model, $campaign);
    }
    
    private function saveTranslations(CampaignEloquentModel $model, Campaign $campaign): void
    {
        $model->translations()->delete(); // Clear existing translations
        
        $translations = $campaign->getAllTranslations();
        
        foreach ($translations['names'] as $locale => $name) {
            $model->translations()->create([
                'locale' => $locale,
                'name' => $name,
                'description' => $translations['descriptions'][$locale] ?? ''
            ]);
        }
    }
    
    private function toDomainModel(CampaignEloquentModel $model): Campaign
    {
        $names = [];
        $descriptions = [];
        
        foreach ($model->translations as $translation) {
            $names[$translation->locale] = $translation->name;
            $descriptions[$translation->locale] = $translation->description;
        }
        
        return Campaign::reconstruct(
            id: CampaignId::fromString($model->id),
            names: $names,
            descriptions: $descriptions,
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

## Frontend Internationalization

### Vue.js Language Switcher Component

```vue
<!-- resources/js/components/LanguageSwitcher.vue -->
<template>
  <div class=\"relative inline-block text-left\">
    <button
      @click=\"toggleDropdown\"
      class=\"inline-flex justify-center w-full rounded-md border border-gray-300 shadow-sm px-4 py-2 bg-white text-sm font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500\"
      :aria-expanded=\"isOpen\"
      aria-haspopup=\"true\"
    >
      <span class=\"mr-2\">{{ currentLanguageFlag }}</span>
      {{ currentLanguageName }}
      <ChevronDownIcon class=\"-mr-1 ml-2 h-5 w-5\" />
    </button>

    <Transition
      enter-active-class=\"transition ease-out duration-100\"
      enter-from-class=\"transform opacity-0 scale-95\"
      enter-to-class=\"transform opacity-100 scale-100\"
      leave-active-class=\"transition ease-in duration-75\"
      leave-from-class=\"transform opacity-100 scale-100\"
      leave-to-class=\"transform opacity-0 scale-95\"
    >
      <div
        v-if=\"isOpen\"
        class=\"origin-top-right absolute right-0 mt-2 w-56 rounded-md shadow-lg bg-white ring-1 ring-black ring-opacity-5 focus:outline-none\"
        role=\"menu\"
      >
        <div class=\"py-1\" role=\"none\">
          <a
            v-for=\"language in availableLanguages\"
            :key=\"language.code\"
            :href=\"getLocalizedUrl(language.code)\"
            class=\"flex items-center px-4 py-2 text-sm text-gray-700 hover:bg-gray-100 hover:text-gray-900\"
            :class=\"{ 'bg-gray-100 text-gray-900': language.code === currentLocale }\"
            role=\"menuitem\"
            @click=\"switchLanguage(language.code)\"
          >
            <span class=\"mr-3\">{{ language.flag }}</span>
            <span>{{ language.name }}</span>
            <span v-if=\"language.code === currentLocale\" class=\"ml-auto\">
              <CheckIcon class=\"h-4 w-4 text-indigo-600\" />
            </span>
          </a>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup lang=\"ts\">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { ChevronDownIcon, CheckIcon } from '@heroicons/vue/24/outline'

interface Language {
  code: string
  name: string
  flag: string
  nativeName: string
}

const isOpen = ref(false)
const currentLocale = ref('en')

const availableLanguages: Language[] = [
  { code: 'en', name: 'English', flag: 'EN', nativeName: 'English' },
  { code: 'fr', name: 'French', flag: 'FR', nativeName: 'Français' },
  { code: 'nl', name: 'Dutch', flag: 'NL', nativeName: 'Nederlands' }
]

const currentLanguage = computed(() => 
  availableLanguages.find(lang => lang.code === currentLocale.value) || availableLanguages[0]
)

const currentLanguageName = computed(() => currentLanguage.value.nativeName)
const currentLanguageFlag = computed(() => currentLanguage.value.flag)

function toggleDropdown() {
  isOpen.value = !isOpen.value
}

function getLocalizedUrl(locale: string): string {
  const currentPath = window.location.pathname
  const pathSegments = currentPath.split('/').filter(segment => segment !== '')
  
  // Remove current locale if present
  if (availableLanguages.some(lang => lang.code === pathSegments[0])) {
    pathSegments.shift()
  }
  
  // Add new locale
  if (locale !== 'en') { // English is default, no prefix needed
    pathSegments.unshift(locale)
  }
  
  return '/' + pathSegments.join('/')
}

function switchLanguage(locale: string) {
  currentLocale.value = locale
  isOpen.value = false
  
  // Store preference
  localStorage.setItem('preferred_locale', locale)
  
  // Navigate to localized URL
  window.location.href = getLocalizedUrl(locale)
}

function handleClickOutside(event: MouseEvent) {
  const target = event.target as HTMLElement
  if (!target.closest('.relative')) {
    isOpen.value = false
  }
}

onMounted(() => {
  // Detect current locale from URL
  const pathSegments = window.location.pathname.split('/').filter(segment => segment !== '')
  const possibleLocale = pathSegments[0]
  
  if (availableLanguages.some(lang => lang.code === possibleLocale)) {
    currentLocale.value = possibleLocale
  }
  
  document.addEventListener('click', handleClickOutside)
})

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside)
})
</script>
```

### Translation Management System

```php
// modules/Shared/Infrastructure/Laravel/Services/TranslationService.php
declare(strict_types=1);

namespace Modules\\Shared\\Infrastructure\\Laravel\\Services;

use Illuminate\\Support\\Facades\\File;
use Illuminate\\Support\\Facades\\Cache;

final readonly class TranslationService
{
    public function __construct(
        private array $supportedLocales = ['en', 'fr', 'nl']
    ) {}
    
    /**
     * Get translation with fallback support and caching
     */
    public function get(string $key, array $replace = [], ?string $locale = null): string
    {
        $locale = $locale ?? app()->getLocale();
        $cacheKey = \"translation.{$locale}.{$key}\";
        
        $translation = Cache::remember($cacheKey, 3600, function () use ($key, $locale) {
            return $this->loadTranslation($key, $locale);
        });
        
        if (!$translation && $locale !== config('app.fallback_locale')) {
            $translation = $this->get($key, $replace, config('app.fallback_locale'));
        }
        
        return $this->replacePlaceholders($translation ?? $key, $replace);
    }
    
    /**
     * Load translation from file system
     */
    private function loadTranslation(string $key, string $locale): ?string
    {
        $keyParts = explode('.', $key);
        $file = array_shift($keyParts);
        
        $filePath = resource_path(\"lang/{$locale}/{$file}.php\");
        
        if (!File::exists($filePath)) {
            return null;
        }
        
        $translations = include $filePath;
        
        return data_get($translations, implode('.', $keyParts));
    }
    
    /**
     * Replace placeholders in translation strings
     */
    private function replacePlaceholders(string $translation, array $replace): string
    {
        foreach ($replace as $key => $value) {
            $translation = str_replace(\":$key\", (string) $value, $translation);
        }
        
        return $translation;
    }
    
    /**
     * Get all translations for a given locale (for frontend hydration)
     */
    public function getAllForLocale(string $locale): array
    {
        $cacheKey = \"translations.all.{$locale}\";
        
        return Cache::remember($cacheKey, 3600, function () use ($locale) {
            $translations = [];
            $langPath = resource_path(\"lang/{$locale}\");
            
            if (!File::isDirectory($langPath)) {
                return [];
            }
            
            $files = File::files($langPath);
            
            foreach ($files as $file) {
                $filename = $file->getFilenameWithoutExtension();
                $fileTranslations = include $file->getPathname();
                
                $translations[$filename] = $fileTranslations;
            }
            
            return $translations;
        });
    }
    
    /**
     * Clear translation cache
     */
    public function clearCache(?string $locale = null): void
    {
        if ($locale) {
            Cache::tags(['translations', $locale])->flush();
        } else {
            Cache::tags(['translations'])->flush();
        }
    }
}
```

## API Localization Strategy

### Localized API Responses

```php
// modules/Shared/Infrastructure/Laravel/Middleware/ApiLocaleMiddleware.php
declare(strict_types=1);

namespace App\\Http\\Middleware;

use Closure;
use Illuminate\\Http\\Request;
use Illuminate\\Support\\Facades\\App;

final readonly class ApiLocaleMiddleware
{
    public function handle(Request $request, Closure $next): mixed
    {
        $locale = $this->determineLocale($request);
        
        if ($locale && $this->isLocaleSupported($locale)) {
            App::setLocale($locale);
        }
        
        return $next($request);
    }
    
    private function determineLocale(Request $request): ?string
    {
        // 1. Check Accept-Language header
        $acceptLanguage = $request->header('Accept-Language');
        if ($acceptLanguage) {
            return $this->parseAcceptLanguageHeader($acceptLanguage);
        }
        
        // 2. Check query parameter
        $queryLocale = $request->query('lang');
        if ($queryLocale) {
            return $queryLocale;
        }
        
        // 3. Check user preference (if authenticated)
        $user = $request->user();
        if ($user && method_exists($user, 'getPreferredLocale')) {
            return $user->getPreferredLocale();
        }
        
        return null;
    }
    
    private function parseAcceptLanguageHeader(string $header): ?string
    {
        $languages = [];
        preg_match_all('/([a-z]{2})(?:-[A-Z]{2})?(?:;q=([0-9.]+))?/', 
            $header, $matches, PREG_SET_ORDER);
        
        foreach ($matches as $match) {
            $lang = $match[1];
            $quality = $match[2] ?? '1.0';
            $languages[$lang] = (float) $quality;
        }
        
        arsort($languages);
        
        foreach (array_keys($languages) as $language) {
            if ($this->isLocaleSupported($language)) {
                return $language;
            }
        }
        
        return null;
    }
    
    private function isLocaleSupported(string $locale): bool
    {
        return in_array($locale, config('app.supported_locales', ['en']), true);
    }
}
```

### Localized API Resource Responses

```php
// modules/Campaign/Infrastructure/Laravel/Resource/CampaignResource.php
declare(strict_types=1);

namespace Modules\\Campaign\\Infrastructure\\Laravel\\Resource;

use Illuminate\\Http\\Resources\\Json\\JsonResource;
use Modules\\Campaign\\Domain\\Model\\Campaign;

final class CampaignResource extends JsonResource
{
    public function toArray($request): array
    {
        /** @var Campaign $campaign */
        $campaign = $this->resource;
        $locale = app()->getLocale();
        
        return [
            'id' => $campaign->getId()->toString(),
            'name' => $campaign->getName($locale),
            'description' => $campaign->getDescription($locale),
            'goal' => [
                'amount' => $campaign->getGoal()->getAmount(),
                'currency' => $campaign->getGoal()->getCurrency(),
                'formatted' => $this->formatMoney($campaign->getGoal(), $locale)
            ],
            'raised' => [
                'amount' => $campaign->getRaised()->getAmount(),
                'currency' => $campaign->getRaised()->getCurrency(),
                'formatted' => $this->formatMoney($campaign->getRaised(), $locale)
            ],
            'status' => $campaign->getStatus()->toString(),
            'progress_percentage' => $campaign->getProgressPercentage(),
            'created_at' => $campaign->getCreatedAt()->format('c'),
            'translations' => $this->when($request->query('include_translations'), function () use ($campaign) {
                return $campaign->getAllTranslations();
            })
        ];
    }
    
    private function formatMoney(Money $money, string $locale): string
    {
        $formatter = new \\NumberFormatter($locale, \\NumberFormatter::CURRENCY);
        
        return $formatter->formatCurrency(
            $money->getAmount(),
            $money->getCurrency()
        );
    }
}
```

## Performance Optimization

### Translation Caching Strategy

```php
// modules/Shared/Infrastructure/Cache/MultilingualCacheService.php
declare(strict_types=1);

namespace Modules\\Shared\\Infrastructure\\Cache;

use Illuminate\\Support\\Facades\\Cache;

final readonly class MultilingualCacheService
{
    public function remember(string $key, int $ttl, callable $callback, ?string $locale = null): mixed
    {
        $locale = $locale ?? app()->getLocale();
        $cacheKey = \"multilingual.{$locale}.{$key}\";
        
        return Cache::remember($cacheKey, $ttl, $callback);
    }
    
    public function forget(string $key, ?string $locale = null): void
    {
        if ($locale) {
            Cache::forget(\"multilingual.{$locale}.{$key}\");
        } else {
            // Clear for all supported locales
            foreach (config('app.supported_locales', ['en']) as $supportedLocale) {
                Cache::forget(\"multilingual.{$supportedLocale}.{$key}\");
            }
        }
    }
    
    public function tags(array $tags, ?string $locale = null): \\Illuminate\\Contracts\\Cache\\Repository
    {
        $locale = $locale ?? app()->getLocale();
        $localizedTags = array_map(fn($tag) => \"{$locale}.{$tag}\", $tags);
        
        return Cache::tags($localizedTags);
    }
}
```

### Database Query Optimization

```sql
-- Optimized queries for multilingual content
-- Get campaign with current locale translation, fallback to default
SELECT 
    c.*,
    COALESCE(ct_current.name, ct_default.name) as name,
    COALESCE(ct_current.description, ct_default.description) as description
FROM campaigns c
LEFT JOIN campaign_translations ct_current ON c.id = ct_current.campaign_id AND ct_current.locale = ?
LEFT JOIN campaign_translations ct_default ON c.id = ct_default.campaign_id AND ct_default.locale = 'en'
WHERE c.status = 'active'
ORDER BY c.created_at DESC;
```

## SEO Optimization

### Hreflang Implementation

```blade
{{-- resources/views/layouts/app.blade.php --}}
@foreach(LaravelLocalization::getSupportedLocales() as $localeCode => $properties)
    <link rel=\"alternate\" hreflang=\"{{ $localeCode }}\" 
          href=\"{{ LaravelLocalization::getLocalizedURL($localeCode, null, [], true) }}\">
@endforeach
<link rel=\"alternate\" hreflang=\"x-default\" 
      href=\"{{ LaravelLocalization::getLocalizedURL('en', null, [], true) }}\">

{{-- Dynamic meta tags based on locale --}}
<title>{{ __('meta.title') }} - {{ config('app.name') }}</title>
<meta name=\"description\" content=\"{{ __('meta.description') }}\">
<meta property=\"og:title\" content=\"{{ __('meta.og_title') }}\">
<meta property=\"og:description\" content=\"{{ __('meta.og_description') }}\">
<meta property=\"og:locale\" content=\"{{ str_replace('-', '_', app()->getLocale()) }}\">
```

## Content Management Strategy

### Translation Workflow

```php
// modules/Admin/Infrastructure/Filament/Resources/TranslationResource.php
declare(strict_types=1);

namespace Modules\\Admin\\Infrastructure\\Filament\\Resources;

use Filament\\Resources\\Resource;
use Filament\\Tables\\Table;
use Filament\\Tables\\Columns\\TextColumn;
use Filament\\Tables\\Filters\\SelectFilter;
use Filament\\Forms\\Form;
use Filament\\Forms\\Components\\TextInput;
use Filament\\Forms\\Components\\Textarea;
use Filament\\Forms\\Components\\Select;

final class TranslationResource extends Resource
{
    protected static ?string $model = Translation::class;
    protected static ?string $navigationIcon = 'heroicon-o-language';
    protected static ?string $navigationGroup = 'Content Management';
    
    public static function form(Form $form): Form
    {
        return $form->schema([
            Select::make('locale')
                ->options([
                    'en' => 'English',
                    'fr' => 'Français', 
                    'nl' => 'Nederlands'
                ])
                ->required(),
                
            TextInput::make('key')
                ->required()
                ->helperText('Translation key (e.g., campaigns.create_button)'),
                
            Textarea::make('value')
                ->required()
                ->rows(3)
                ->helperText('Translation value in the selected language'),
                
            Select::make('status')
                ->options([
                    'draft' => 'Draft',
                    'review' => 'Needs Review',
                    'approved' => 'Approved'
                ])
                ->default('draft')
        ]);
    }
    
    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('locale')
                    ->badge()
                    ->color(fn (string $state): string => match ($state) {
                        'en' => 'success',
                        'fr' => 'info',
                        'nl' => 'warning',
                    }),
                    
                TextColumn::make('key')
                    ->searchable()
                    ->sortable(),
                    
                TextColumn::make('value')
                    ->limit(50)
                    ->tooltip(function (TextColumn $column): ?string {
                        $state = $column->getState();
                        return strlen($state) > 50 ? $state : null;
                    }),
                    
                TextColumn::make('status')
                    ->badge()
                    ->color(fn (string $state): string => match ($state) {
                        'draft' => 'gray',
                        'review' => 'warning',
                        'approved' => 'success',
                    }),
                    
                TextColumn::make('updated_at')
                    ->dateTime()
                    ->sortable()
            ])
            ->filters([
                SelectFilter::make('locale')
                    ->options([
                        'en' => 'English',
                        'fr' => 'Français',
                        'nl' => 'Nederlands'
                    ]),
                    
                SelectFilter::make('status')
                    ->options([
                        'draft' => 'Draft',
                        'review' => 'Needs Review', 
                        'approved' => 'Approved'
                    ])
            ])
            ->defaultSort('updated_at', 'desc');
    }
}
```

## Testing Internationalization

### Localization Testing Strategy

```php
// tests/Feature/LocalizationTest.php
declare(strict_types=1);

namespace Tests\\Feature;

use Tests\\TestCase;
use Illuminate\\Foundation\\Testing\\RefreshDatabase;

final class LocalizationTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_homepage_redirects_to_browser_language(): void
    {
        $response = $this->get('/', [
            'Accept-Language' => 'fr-FR,fr;q=0.9,en;q=0.8'
        ]);
        
        $response->assertRedirect('/fr');
    }
    
    public function test_campaigns_display_correct_language_content(): void
    {
        $campaign = Campaign::factory()->create();
        $campaign->setName('Clean Water Initiative', 'en');
        $campaign->setName('Initiative Eau Propre', 'fr');
        $campaign->save();
        
        // Test English
        $response = $this->get('/en/campaigns/' . $campaign->id);
        $response->assertSee('Clean Water Initiative');
        
        // Test French
        $response = $this->get('/fr/campaigns/' . $campaign->id);
        $response->assertSee('Initiative Eau Propre');
    }
    
    public function test_api_respects_accept_language_header(): void
    {
        $campaign = Campaign::factory()->create();
        
        $response = $this->getJson('/api/campaigns/' . $campaign->id, [
            'Accept-Language' => 'fr'
        ]);
        
        $response->assertStatus(200)
                 ->assertJsonFragment([
                     'name' => $campaign->getName('fr')
                 ]);
    }
}
```

## Strategic Benefits

### For Global Enterprise Deployment

1. **Market Readiness**: Platform ready for international markets
2. **User Experience**: Native language experience increases adoption
3. **SEO Benefits**: Better search engine rankings in local markets
4. **Compliance**: Meets international localization requirements

### For Development Teams

1. **Scalable Architecture**: Easy to add new languages
2. **Performance Optimized**: Caching prevents translation lookup overhead
3. **Content Management**: Admin interface for translation management
4. **Testing Coverage**: Comprehensive localization testing

### Technical Leadership Demonstration

1. **Strategic Planning**: Proactive internationalization architecture
2. **Performance Consideration**: Caching and optimization strategies
3. **User Experience Focus**: Intelligent language detection and persistence
4. **Enterprise Integration**: URL-based routing for corporate environments

This internationalization architecture demonstrates technical leadership through forward-thinking global deployment planning, comprehensive user experience design, and enterprise-grade implementation strategies.

---

**Developed and Maintained by [Go2digit.al](https://go2digit.al)**

Specialized in enterprise-grade applications with focus on scalability, security, and maintainability.

Copyright 2025 Go2digit.al - All Rights Reserved