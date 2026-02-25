# Security in Laravel

## Authentication & Authorization

### Gates and Policies

```php
// Granular authorization with policies
final class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->customer_id || $user->hasRole('admin');
    }

    public function update(User $user, Order $order): bool
    {
        return $user->id === $order->customer_id && $order->status === 'placed';
    }
}

// Laravel 11+: Auto-discovered when OrderPolicy in app/Policies/ maps to Order model.
// Laravel 10: Register in AuthServiceProvider: protected $policies = [Order::class => OrderPolicy::class];

// Controller usage
public function update(Request $request, Order $order): JsonResponse
{
    $this->authorize('update', $order);  // throws 403 if denied
    // ...
}

// Blade usage
@can('update', $order)
    <button>Edit</button>
@endcan
```

### Role-Based Access (spatie/laravel-permission)

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
```

```php
// Roles and permissions
$user->assignRole('editor');
$user->givePermissionTo('publish articles');

// Check
$user->hasRole('editor');
$user->can('publish articles');

// Middleware
Route::middleware(['auth', 'role:admin'])->group(fn () => /* ... */);
Route::middleware(['auth', 'permission:publish articles'])->group(fn () => /* ... */);
```

## Input Validation & Sanitization

```php
// Always validate at the boundary — never trust input
final class CreateOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items'            => ['required', 'array', 'min:1'],
            'items.*.sku'      => ['required', 'string', 'alpha_dash', 'max:50'],
            'items.*.quantity' => ['required', 'integer', 'min:1', 'max:999'],
            'coupon_code'      => ['nullable', 'string', 'regex:/^[A-Z0-9\-]{4,20}$/'],
            'notes'            => ['nullable', 'string', 'max:500'],
        ];
    }
}
```

**Never pass raw request data to models:**
```php
// ❌ NEVER
Order::create($request->all());

// ✅ Use $fillable + validated()
Order::create($request->validated());
```

## SQL Injection Prevention

Laravel's query builder and Eloquent use PDO parameter binding by default. Danger zones:

```php
// ❌ DANGEROUS — never interpolate user input into raw SQL
DB::select("SELECT * FROM orders WHERE status = '{$request->status}'");

// ✅ SAFE — always use bindings
DB::select('SELECT * FROM orders WHERE status = ?', [$request->status]);
DB::table('orders')->where('status', $request->status)->get();

// ❌ DANGEROUS — orderBy with user input
->orderByRaw($request->sort_column);

// ✅ SAFE — whitelist allowed columns
$allowed = ['created_at', 'total_cents', 'status'];
$column = in_array($request->sort, $allowed) ? $request->sort : 'created_at';
->orderBy($column, $request->direction === 'asc' ? 'asc' : 'desc');
```

## XSS Prevention

Blade auto-escapes `{{ }}`. Danger zones:

```blade
{{-- ✅ Safe — escaped --}}
{{ $user->name }}

{{-- ❌ DANGEROUS — only use for trusted HTML --}}
{!! $user->bio !!}
```

**Purify user HTML (when rich text is needed):**
```bash
composer require ezyang/htmlpurifier
```

```php
$clean = (new HTMLPurifier())->purify($request->body);
```

**Content Security Policy:**
```php
// Middleware to add CSP header
public function handle(Request $request, Closure $next): mixed
{
    $response = $next($request);
    $nonce = base64_encode(random_bytes(16));
    $request->attributes->set('csp_nonce', $nonce);

    $response->headers->set('Content-Security-Policy',
        "default-src 'self'; " .
        "script-src 'self' 'nonce-{$nonce}'; " .
        "style-src 'self' 'nonce-{$nonce}'; " .
        "img-src 'self' data: https:; " .
        "connect-src 'self' wss:;"
    );
    return $response;
}
```

## CSRF Protection

Laravel's `VerifyCsrfToken` middleware protects all non-GET routes. For APIs using Sanctum SPA authentication, CSRF cookie must be fetched:

```tsx
// Before first API request from SPA
await axios.get('/sanctum/csrf-cookie')
```

Exclude only truly stateless API routes:
```php
// app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'api/webhooks/*',   // external webhook receivers
];
```

## Security Headers

```php
// Middleware: add security headers to all responses
public function handle(Request $request, Closure $next): mixed
{
    $response = $next($request);
    $response->headers->set('X-Frame-Options', 'DENY');
    $response->headers->set('X-Content-Type-Options', 'nosniff');
    $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    $response->headers->set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
    $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    return $response;
}
```

## Encryption & Sensitive Data

```php
// Encrypt sensitive model attributes at rest
use Illuminate\Database\Eloquent\Casts\AsEncryptedString;

protected $casts = [
    'ssn'           => AsEncryptedString::class,
    'bank_account'  => AsEncryptedString::class,
    'api_secret'    => AsEncryptedString::class,
];

// Manual encryption
$encrypted = Crypt::encryptString($secret);
$plain     = Crypt::decryptString($encrypted);

// Hash passwords (never encrypt)
$hashed = Hash::make($password);
Hash::check($password, $hashed);
```

**Never store in plaintext:** passwords, API keys, tokens, SSNs, credit card data, health records.

## Mass Assignment Protection

```php
// ✅ Use $fillable (allowlist) — safer default
protected $fillable = ['name', 'email', 'address'];

// ❌ Avoid $guarded = [] (blocks nothing)
// ❌ Never use Model::unguard() in production
```

## File Upload Security

```php
public function upload(Request $request): JsonResponse
{
    $request->validate([
        'file' => [
            'required',
            'file',
            'max:10240',  // 10MB
            'mimes:pdf,png,jpg,webp',  // NEVER allow php, sh, exe
        ],
    ]);

    // Always store outside public web root
    $path = $request->file('file')->store('uploads', 'private');

    // Serve via signed URL — not directly
    return response()->json([
        'url' => Storage::disk('private')->temporaryUrl($path, now()->addHours(1)),
    ]);
}
```

## Rate Limiting & Brute Force Protection

```php
// Login brute force protection
RateLimiter::for('login', function (Request $request) {
    return Limit::perMinute(5)->by($request->input('email') . '|' . $request->ip());
});

// API rate limiting with different tiers
RateLimiter::for('api', function (Request $request) {
    if ($request->user()?->hasRole('premium')) {
        return Limit::perMinute(1000)->by($request->user()->id);
    }
    return Limit::perMinute(60)->by($request->user()?->id ?? $request->ip());
});
```

## Audit Logging (spatie/laravel-activitylog)

```bash
composer require spatie/laravel-activitylog
```

```php
// Auto-log model changes
use Spatie\Activitylog\Traits\LogsActivity;
use Spatie\Activitylog\LogOptions;

class Order extends Model
{
    use LogsActivity;

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logOnly(['status', 'total_cents'])
            ->logOnlyDirty()
            ->dontSubmitEmptyLogs();
    }
}

// Manual log
activity('security')
    ->causedBy($user)
    ->performedOn($order)
    ->withProperties(['ip' => request()->ip()])
    ->log('Accessed sensitive order data');
```

## Dependency Security

```bash
# Check for known vulnerabilities in Composer packages
composer audit

# Check npm packages
npm audit

# Add to CI pipeline — fail on high severity
composer audit --format=plain
npm audit --audit-level=high
```

## OWASP Top 10 Checklist

| Risk | Laravel Mitigation |
|---|---|
| Broken Access Control | Policies, Gates, route middleware |
| Cryptographic Failures | `AsEncryptedString`, `Hash::make()`, HTTPS |
| Injection | Eloquent bindings, validated input, whitelist columns |
| Insecure Design | `$fillable`, FormRequests, service layer validation |
| Security Misconfiguration | Security headers middleware, `APP_DEBUG=false` in prod |
| Vulnerable Components | `composer audit`, `npm audit` in CI |
| Authentication Failures | Rate limiting, Sanctum, 2FA |
| Integrity Failures | Signed URLs, HMAC webhook validation |
| Logging Failures | Activitylog, structured logging, alerting |
| SSRF | Validate URLs, block internal IP ranges in outbound HTTP |

**SSRF prevention for outbound requests:**
```php
Http::withOptions([
    'allow_redirects' => false,
    'on_headers' => function ($response, $requestOptions) {
        $host = parse_url($requestOptions['base_uri'], PHP_URL_HOST);
        if (filter_var(gethostbyname($host), FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
            throw new \RuntimeException('SSRF attempt blocked: private IP range');
        }
    },
])->get($userProvidedUrl);
```
