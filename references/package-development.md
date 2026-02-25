# Laravel Package Development

## Package Skeleton

```bash
composer create-project spatie/laravel-package-tools-skeleton my-package
cd my-package
```

Or scaffold manually:

```
my-package/
├── src/
│   ├── MyPackageServiceProvider.php
│   ├── Facades/
│   │   └── MyPackage.php
│   ├── Commands/
│   ├── Http/
│   │   ├── Controllers/
│   │   └── Middleware/
│   ├── Models/
│   └── MyPackage.php          # Main class
├── config/
│   └── my-package.php
├── database/
│   └── migrations/
├── resources/
│   ├── views/
│   └── lang/
├── routes/
│   └── web.php
├── tests/
│   ├── TestCase.php
│   └── Feature/
├── composer.json
└── README.md
```

---

## Service Provider

```php
// src/MyPackageServiceProvider.php
use Spatie\LaravelPackageTools\Package;
use Spatie\LaravelPackageTools\PackageServiceProvider;

final class MyPackageServiceProvider extends PackageServiceProvider
{
    public function configurePackage(Package $package): void
    {
        $package
            ->name('my-package')
            ->hasConfigFile()                               // config/my-package.php
            ->hasMigrations(['create_my_package_table'])   // auto-publishable
            ->hasViews('my-package')                       // namespace: my-package::
            ->hasTranslations()
            ->hasCommands([
                MyPackageCommand::class,
            ])
            ->hasRoutes(['web', 'api']);
    }

    public function packageRegistered(): void
    {
        // Bindings available before boot
        $this->app->singleton(MyPackageManager::class, function ($app) {
            return new MyPackageManager($app['config']['my-package']);
        });
    }

    public function packageBooted(): void
    {
        // Hooks that run after all providers are booted
        MyPackage::observe(MyPackageObserver::class);

        // Macro on existing Laravel classes
        \Illuminate\Support\Collection::macro('toMyFormat', function () {
            return $this->map(fn ($item) => MyPackage::transform($item));
        });
    }
}
```

## Facade

```php
// src/Facades/MyPackage.php
use Illuminate\Support\Facades\Facade;

/**
 * @method static string process(string $input)
 * @method static array analyze(mixed $data)
 * @see \Vendor\MyPackage\MyPackage
 */
final class MyPackage extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return MyPackageManager::class;
    }
}
```

## composer.json

```json
{
    "name": "your-vendor/my-package",
    "description": "A brief description of your package",
    "keywords": ["laravel", "my-package"],
    "license": "MIT",
    "authors": [{"name": "Your Name", "email": "you@example.com"}],
    "require": {
        "php": "^8.2",
        "illuminate/support": "^11.0|^12.0"
    },
    "require-dev": {
        "orchestra/testbench": "^9.0",
        "pestphp/pest": "^3.0",
        "pestphp/pest-plugin-laravel": "^3.0"
    },
    "autoload": {
        "psr-4": {"Vendor\\MyPackage\\": "src/"}
    },
    "autoload-dev": {
        "psr-4": {"Vendor\\MyPackage\\Tests\\": "tests/"}
    },
    "extra": {
        "laravel": {
            "providers": ["Vendor\\MyPackage\\MyPackageServiceProvider"],
            "aliases": {"MyPackage": "Vendor\\MyPackage\\Facades\\MyPackage"}
        }
    },
    "config": {
        "sort-packages": true,
        "allow-plugins": {"pestphp/pest-plugin": true}
    }
}
```

---

## Testing with Orchestra Testbench

Testbench provides a full Laravel application environment for package tests:

```php
// tests/TestCase.php
use Orchestra\Testbench\TestCase as OrchestraTestCase;

abstract class TestCase extends OrchestraTestCase
{
    protected function getPackageProviders($app): array
    {
        return [MyPackageServiceProvider::class];
    }

    protected function getPackageAliases($app): array
    {
        return ['MyPackage' => MyPackageFacade::class];
    }

    protected function defineEnvironment($app): void
    {
        $app['config']->set('my-package.option', 'test-value');
        $app['config']->set('database.default', 'testing');
    }

    protected function defineDatabaseMigrations(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
    }
}
```

```php
// tests/Feature/MyPackageTest.php
uses(TestCase::class);

it('processes input correctly', function () {
    $result = MyPackage::process('hello world');
    expect($result)->toBe('HELLO WORLD');
});

it('registers config correctly', function () {
    expect(config('my-package.option'))->toBe('test-value');
});
```

---

## Config File

```php
// config/my-package.php
return [
    /*
    |--------------------------------------------------------------------------
    | Default Driver
    |--------------------------------------------------------------------------
    | Supported: "sync", "async", "custom"
    */
    'driver' => env('MY_PACKAGE_DRIVER', 'sync'),

    'options' => [
        'timeout'  => env('MY_PACKAGE_TIMEOUT', 30),
        'retries'  => env('MY_PACKAGE_RETRIES', 3),
    ],

    'drivers' => [
        'custom' => [
            'class' => null,  // Override with your own implementation
        ],
    ],
];
```

## Migrations

```php
// database/migrations/create_my_package_table.php.stub
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('my_package_records', function (Blueprint $table) {
            $table->id();
            $table->morphs('subject');         // polymorphic relationship
            $table->string('type');
            $table->json('payload')->nullable();
            $table->timestamps();

            $table->index(['subject_type', 'subject_id', 'type']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('my_package_records');
    }
};
```

---

## Artisan Commands

```php
// src/Commands/MyPackageCommand.php
use Illuminate\Console\Command;

final class MyPackageCommand extends Command
{
    protected $signature   = 'my-package:run
                                {--dry-run : Show what would happen without doing it}
                                {--limit=100 : Number of records to process}';
    protected $description = 'Run the my-package processor';

    public function handle(MyPackageManager $manager): int
    {
        $limit  = (int) $this->option('limit');
        $dryRun = $this->option('dry-run');

        $this->info("Processing up to {$limit} records...");

        $bar = $this->output->createProgressBar($limit);
        $bar->start();

        $count = $manager->process(limit: $limit, dryRun: $dryRun, onProgress: fn () => $bar->advance());

        $bar->finish();
        $this->newLine();
        $this->info("Done! Processed {$count} records.");

        return self::SUCCESS;
    }
}
```

---

## Traits for Consumer Models

```php
// src/Concerns/HasMyPackage.php
trait HasMyPackage
{
    public static function bootHasMyPackage(): void
    {
        static::created(fn ($model) => MyPackage::onCreated($model));
        static::deleted(fn ($model) => MyPackage::onDeleted($model));
    }

    public function myPackageRecords(): HasMany
    {
        return $this->hasMany(MyPackageRecord::class, 'subject_id')
            ->where('subject_type', static::class);
    }

    public function processWithMyPackage(array $options = []): MyPackageResult
    {
        return MyPackage::processModel($this, $options);
    }
}

// Consumer usage:
class Order extends Model
{
    use HasMyPackage;
}
$order->processWithMyPackage(['option' => 'value']);
```

---

## Events & Hooks

```php
// src/Events/MyPackageProcessed.php
final class MyPackageProcessed
{
    public function __construct(
        public readonly Model $subject,
        public readonly MyPackageResult $result,
    ) {}
}

// Allow consumers to hook in via config callbacks
// config/my-package.php
'after_processing' => null, // callable

// In package:
$callback = config('my-package.after_processing');
if (is_callable($callback)) {
    $callback($subject, $result);
}

event(new MyPackageProcessed($subject, $result));
```

---

## Versioning & Changelog

Follow **Semantic Versioning** (semver.org):
- `MAJOR`: Breaking change (new major version requirement, removed methods, signature change)
- `MINOR`: New feature, backward compatible
- `PATCH`: Bug fix, backward compatible

**CHANGELOG.md template:**
```markdown
## [Unreleased]

## [1.1.0] - 2026-02-01
### Added
- Support for Laravel 12
- New `processAsync()` method

### Fixed
- Memory leak when processing large datasets

## [1.0.0] - 2026-01-01
### Initial release
```

---

## GitHub Actions CI

```yaml
# .github/workflows/run-tests.yml
name: Run Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        php: ['8.2', '8.3', '8.4']
        laravel: ['11.*', '12.*']
        stability: [prefer-lowest, prefer-stable]

    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          coverage: xdebug

      - run: composer require "laravel/framework:${{ matrix.laravel }}" --no-interaction --no-update
      - run: composer update --${{ matrix.stability }} --prefer-dist --no-interaction

      - run: vendor/bin/pest --coverage --min=90
```

---

## Publishing to Packagist

1. Push to GitHub (public repo)
2. Register at [packagist.org](https://packagist.org) → Submit → enter GitHub URL
3. Add GitHub webhook for auto-updates:
   - Packagist → Profile → Show API Token
   - GitHub repo → Settings → Webhooks → Add webhook
   - URL: `https://packagist.org/api/github?username=YOUR_USERNAME`
   - Secret: your Packagist API token
4. Tag a release: `git tag v1.0.0 && git push --tags`
5. Packagist auto-updates on each tag push

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Coupling to specific Laravel version | Use `illuminate/support` not `laravel/framework` |
| Hardcoded config keys | Always read from `config('my-package.*')` |
| Migrations run automatically | Use `loadMigrationsFrom` in tests only; `publishesMigrations` for production |
| No PHP version matrix in CI | Test against all supported PHP versions |
| Breaking changes without major bump | Follow semver strictly |
| Missing `$this->callAfterResolving` for deferred boot | Use `packageBooted()` for hooks that need full app |
