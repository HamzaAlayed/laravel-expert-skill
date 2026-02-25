# DDD & Clean Architecture in Laravel

## Directory Layout

```
src/
  Billing/                          # Bounded context
    Domain/
      Model/
        Invoice.php                 # Aggregate root
        InvoiceId.php               # Value object
        Money.php                   # Value object
        InvoiceStatus.php           # Enum
      Event/
        InvoiceWasIssued.php        # Domain event
        InvoiceWasPaid.php
      Repository/
        InvoiceRepository.php       # Interface (port)
      Service/
        InvoicingService.php        # Domain service
    Application/
      Command/
        IssueInvoice.php            # Command DTO (readonly)
        IssueInvoiceHandler.php
      Query/
        GetInvoiceSummary.php       # Query DTO
        GetInvoiceSummaryHandler.php
        InvoiceSummaryDto.php       # Read DTO (not Eloquent)
    Infrastructure/
      Persistence/
        EloquentInvoiceRepository.php  # Adapter
        InvoiceModel.php               # Eloquent model
      Http/
        InvoiceController.php
        InvoiceResource.php
```

## Aggregate Root

```php
// Domain/Model/Invoice.php
final class Invoice
{
    private array $domainEvents = [];

    private function __construct(
        private readonly InvoiceId $id,
        private readonly CustomerId $customerId,
        private Money $total,
        private InvoiceStatus $status,
    ) {}

    public static function issue(
        InvoiceId $id,
        CustomerId $customerId,
        Money $total,
    ): self {
        $invoice = new self($id, $customerId, $total, InvoiceStatus::Draft);
        $invoice->record(new InvoiceWasIssued($id, $customerId, $total));
        return $invoice;
    }

    public function pay(): void
    {
        if ($this->status !== InvoiceStatus::Draft) {
            throw new \DomainException('Only draft invoices can be paid.');
        }
        $this->status = InvoiceStatus::Paid;
        $this->record(new InvoiceWasPaid($this->id));
    }

    public function releaseEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    private function record(object $event): void
    {
        $this->domainEvents[] = $event;
    }
}
```

## Value Object

```php
// Domain/Model/Money.php
final class Money
{
    public function __construct(
        public readonly int $amount,      // stored in cents
        public readonly string $currency, // ISO 4217
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative.');
        }
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Cannot add different currencies.');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }
}
```

## Repository Interface (Port)

```php
// Domain/Repository/InvoiceRepository.php
interface InvoiceRepository
{
    public function nextId(): InvoiceId;
    public function findById(InvoiceId $id): Invoice;
    public function save(Invoice $invoice): void;
}
```

## Eloquent Adapter (Infrastructure)

```php
// Infrastructure/Persistence/EloquentInvoiceRepository.php
final class EloquentInvoiceRepository implements InvoiceRepository
{
    public function nextId(): InvoiceId
    {
        return InvoiceId::fromString((string) Str::uuid());
    }

    public function findById(InvoiceId $id): Invoice
    {
        $model = InvoiceModel::findOrFail($id->toString());
        return $this->toDomain($model);
    }

    public function save(Invoice $invoice): void
    {
        $data = $this->toModel($invoice);
        InvoiceModel::updateOrCreate(['id' => $data['id']], $data);

        // Dispatch domain events through Laravel's event system
        foreach ($invoice->releaseEvents() as $event) {
            event($event);
        }
    }

    private function toDomain(InvoiceModel $model): Invoice { /* ... */ }
    private function toModel(Invoice $invoice): array { /* ... */ }
}
```

## Application Service (Command Handler)

```php
// Application/Command/IssueInvoiceHandler.php
final class IssueInvoiceHandler
{
    public function __construct(
        private readonly InvoiceRepository $invoices,
    ) {}

    public function handle(IssueInvoice $command): void
    {
        $invoice = Invoice::issue(
            $this->invoices->nextId(),
            CustomerId::fromString($command->customerId),
            new Money($command->amountInCents, $command->currency),
        );

        $this->invoices->save($invoice);
    }
}
```

## Ports and Adapters Mapping

| Hexagonal Concept | Laravel Equivalent |
|---|---|
| Primary port (driving) | Controller, Artisan command, Job |
| Primary adapter | `IssueInvoiceHandler` injected into controller |
| Secondary port (driven) | `InvoiceRepository` interface |
| Secondary adapter | `EloquentInvoiceRepository` |
| Application core | Domain + Application layers |

## Event-Listener Wiring

```php
// App\Providers\EventServiceProvider.php (manual registration for custom locations)
protected $listen = [
    InvoiceWasPaid::class => [
        SendPaymentConfirmationEmail::class,  // Infrastructure listener
        UpdateAccountingSystem::class,
    ],
];
```

Laravel 11+ auto-discovers listeners in `app/Listeners/` by type-hint. Use `$listen` when listeners live outside that path (e.g. Infrastructure). Domain events are plain PHP objects. Domain never imports listeners.

## Binding in AppServiceProvider

```php
$this->app->bind(InvoiceRepository::class, EloquentInvoiceRepository::class);
$this->app->bind(IssueInvoiceHandler::class); // auto-resolved via DI
```

## When to Extract a Bounded Context

**Extract when:**
- The subdomain has its own ubiquitous language distinct from other subdomains
- Two teams will own the area independently
- The data model is fundamentally different from other domains

**Keep inline when:**
- Fewer than 5-7 domain models
- Same team owns everything
- Simple CRUD with no complex business rules

## Architecture Fitness Tests (Pest)

```php
// tests/Architecture/BillingTest.php
use Illuminate\Database\Eloquent\Model;

test('domain layer has no Eloquent dependencies')
    ->expect('Src\Billing\Domain')
    ->not->toUse(Model::class);

test('domain layer has no Laravel facade dependencies')
    ->expect('Src\Billing\Domain')
    ->not->toUse('Illuminate\Support\Facades');

test('application layer has no Eloquent dependencies')
    ->expect('Src\Billing\Application')
    ->not->toUse(Model::class);
```

Run `php artisan test --filter=Architecture` in CI to enforce layering.
