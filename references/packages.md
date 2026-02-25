# Essential Laravel Packages & Ecosystem

## Spatie Collection (Most-Used)

### spatie/laravel-permission — Roles & Permissions
```bash
composer require spatie/laravel-permission
```
See `references/security.md` for full usage.

### spatie/laravel-medialibrary — File Attachments
```bash
composer require spatie/laravel-medialibrary
php artisan vendor:publish --provider="Spatie\MediaLibrary\MediaLibraryServiceProvider" --tag="medialibrary-migrations"
```

```php
class Product extends Model implements HasMedia
{
    use InteractsWithMedia;

    public function registerMediaCollections(): void
    {
        $this->addMediaCollection('images')
            ->acceptsMimeTypes(['image/jpeg', 'image/png', 'image/webp'])
            ->withResponsiveImages();  // auto-generates srcset sizes

        $this->addMediaCollection('documents')
            ->singleFile()            // only one document at a time
            ->useDisk('private');
    }

    public function registerMediaConversions(?Media $media = null): void
    {
        $this->addMediaConversion('thumb')->width(300)->height(300)->nonQueued();
        $this->addMediaConversion('large')->width(1200)->performOnCollections('images');
    }
}

// Usage
$product->addMediaFromRequest('image')->toMediaCollection('images');
$product->getFirstMediaUrl('images', 'thumb');
$product->getMedia('images');        // all images
```

### spatie/laravel-query-builder — API Filtering
```bash
composer require spatie/laravel-query-builder
```

```php
// Automatic filter/sort/include from query string
// GET /orders?filter[status]=placed&sort=-created_at&include=customer,items
$orders = QueryBuilder::for(Order::class)
    ->allowedFilters([
        'status',
        AllowedFilter::exact('customer_id'),
        AllowedFilter::scope('created_after', 'createdAfter'),
        AllowedFilter::custom('search', new OrderSearchFilter()),
    ])
    ->allowedSorts(['created_at', 'total_cents', AllowedSort::field('customer', 'customer_id')])
    ->allowedIncludes(['customer', 'items', 'items.product'])
    ->paginate(25);
```

### spatie/laravel-data — Typed Data Objects
```bash
composer require spatie/laravel-data
```

```php
// Validated, typed DTOs that work as API resources too
#[TypeScript]
final class OrderData extends Data
{
    public function __construct(
        public readonly string $id,
        public readonly OrderStatus $status,
        public readonly MoneyData $total,
        #[DataCollectionOf(OrderItemData::class)]
        public readonly DataCollection $items,
        public readonly Carbon $createdAt,
    ) {}

    public static function fromModel(Order $order): self
    {
        return new self(
            id:        $order->id,
            status:    $order->status,
            total:     MoneyData::fromValueObject($order->total),
            items:     OrderItemData::collect($order->items),
            createdAt: $order->created_at,
        );
    }
}

// Use as FormRequest, API Resource, and DTO simultaneously
Route::post('/orders', function (OrderData $data) { /* validated + typed */ });
```

### spatie/laravel-activitylog — Audit Trail
See `references/security.md` for full usage.

### spatie/laravel-backup — Database Backups
```php
// config/backup.php
'backup' => [
    'source' => ['databases' => ['mysql']],
    'destination' => ['disks' => ['s3']],
    'notifications' => ['mail' => ['to' => 'admin@example.com']],
],

// Schedule
$schedule->command('backup:run')->daily()->at('02:00');
$schedule->command('backup:clean')->daily()->at('01:00');
```

---

## Laravel Official Packages

### Laravel Scout — Full-Text Search
```bash
composer require laravel/scout
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle  # Meilisearch driver
```

```php
// config/scout.php
'driver' => env('SCOUT_DRIVER', 'meilisearch'),

// Model
class Product extends Model
{
    use Searchable;

    public function toSearchableArray(): array
    {
        return [
            'id'          => $this->id,
            'name'        => $this->name,
            'description' => $this->description,
            'category'    => $this->category->name,
            'price_cents' => $this->price_cents,
        ];
    }

    // Only index active products
    public function shouldBeSearchable(): bool { return $this->is_active; }
}

// Search
Product::search('wireless headphones')
    ->where('category', 'electronics')
    ->orderBy('price_cents', 'asc')
    ->paginate(25);
```

**Meilisearch config (config/scout.php):**
```php
'meilisearch' => [
    'host' => env('MEILISEARCH_HOST', 'http://localhost:7700'),
    'key'  => env('MEILISEARCH_KEY'),
    'index-settings' => [
        Product::class => [
            'filterableAttributes' => ['category', 'price_cents'],
            'sortableAttributes'   => ['price_cents', 'name'],
            'searchableAttributes' => ['name', 'description', 'category'],
        ],
    ],
],
```

### Laravel Cashier (Stripe) — Subscriptions & Billing
```bash
composer require laravel/cashier
php artisan cashier:install
```

```php
// Model
class User extends Authenticatable
{
    use Billable;
}

// Subscribe
$user->newSubscription('default', 'price_monthly_pro')
    ->trialDays(14)
    ->create($paymentMethod);

// Check
$user->subscribed('default');
$user->onTrial('default');
$user->subscription('default')->cancelled();

// One-time charge
$user->charge(2999, $paymentMethod, ['description' => 'One-time purchase']);

// Portal (Stripe-hosted billing management)
return $user->redirectToBillingPortal(route('dashboard'));
```

### Laravel Pennant — Feature Flags
```bash
composer require laravel/pennant
php artisan pennant:install
```

```php
// Define feature
Feature::define('new-checkout', function (User $user) {
    return $user->isInBetaGroup() || $user->hasRole('admin');
});

// Percentage rollout
Feature::define('redesign', fn () => rand(1, 100) <= 20);  // 20% rollout

// Check
Feature::active('new-checkout');                    // current user
Feature::for($user)->active('new-checkout');        // specific user

// Blade
@feature('new-checkout')
    <x-new-checkout />
@else
    <x-classic-checkout />
@endfeature
```

### Laravel Telescope — Debugging Dashboard
```bash
composer require laravel/telescope --dev
php artisan telescope:install
```

Captures: requests, queries, jobs, events, notifications, mail, cache, exceptions, logs.

**Gate for production:**
```php
protected function gate(): void
{
    Gate::define('viewTelescope', fn ($user) => $user->hasRole('developer'));
}
```

### Laravel Horizon — Queue Dashboard
See `references/scaling.md` for scaling configuration.

```php
// config/horizon.php — metrics retention
'trim' => [
    'recent'  => 60,    // minutes
    'pending' => 60,
    'completed' => 60,
    'failed'  => 10080, // 1 week
],
```

---

## Package Development

```bash
# Generate package skeleton
composer create-project spatie/laravel-package-tools-skeleton my-package
```

```php
// src/MyPackageServiceProvider.php
final class MyPackageServiceProvider extends PackageServiceProvider
{
    public function configurePackage(Package $package): void
    {
        $package
            ->name('my-package')
            ->hasConfigFile()
            ->hasMigrations(['create_my_table'])
            ->hasViews()
            ->hasTranslations()
            ->hasCommands([MyCommand::class]);
    }

    public function packageBooted(): void
    {
        // Register bindings after package is booted
        $this->app->bind(MyInterface::class, MyImplementation::class);
    }
}
```

**Publish to Packagist:**
1. Push to GitHub
2. Register at `packagist.org`
3. Add GitHub webhook for auto-updates

---

## Useful Third-Party Packages

| Package | Purpose |
|---|---|
| `barryvdh/laravel-ide-helper` | PHPStorm/VSCode autocomplete for facades |
| `dedoc/scramble` | Auto-generate OpenAPI docs from code |
| `lorisleiva/laravel-actions` | Single-class actions (controller + job + listener) |
| `tightenco/ziggy` | Laravel routes in JavaScript |
| `opcodesio/log-viewer` | Beautiful log viewer UI |
| `wire-elements/modal` | Livewire modal component |
| `maatwebsite/excel` | Excel/CSV import & export |
| `barryvdh/laravel-dompdf` | PDF generation |
| `intervention/image` | Image manipulation |
| `staudenmeir/eloquent-has-many-deep` | Deep nested Eloquent relationships |
| `spatie/laravel-settings` | Typed application settings |
| `spatie/laravel-sluggable` | Auto-generate URL slugs |
| `spatie/laravel-translatable` | Multi-language model attributes |
| `spatie/laravel-tags` | Tagging models |
| `spatie/laravel-responsecache` | Full-page response caching |
