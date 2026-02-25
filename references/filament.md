# Filament 3 — Admin Panels in Laravel

## Installation

```bash
composer require filament/filament:"^3.0"
php artisan filament:install --panels
```

Creates `app/Providers/Filament/AdminPanelProvider.php` — the panel configuration hub.

## Panel Configuration

```php
// app/Providers/Filament/AdminPanelProvider.php
public function panel(Panel $panel): Panel
{
    return $panel
        ->default()
        ->id('admin')
        ->path('admin')
        ->login()
        ->colors(['primary' => Color::Indigo])
        ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
        ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
        ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
        ->middleware([EncryptCookies::class, VerifyCsrfToken::class])
        ->authMiddleware([Authenticate::class])
        ->navigation(fn (NavigationBuilder $nav) => $nav
            ->groups([
                NavigationGroup::make('Shop')->items([
                    ...OrderResource::getNavigationItems(),
                    ...ProductResource::getNavigationItems(),
                ]),
            ])
        );
}
```

## Resource (CRUD)

```php
php artisan make:filament-resource Order --generate
```

```php
// app/Filament/Resources/OrderResource.php
final class OrderResource extends Resource
{
    protected static ?string $model = Order::class;
    protected static ?string $navigationIcon = 'heroicon-o-shopping-cart';
    protected static ?string $navigationGroup = 'Shop';
    protected static ?int $navigationSort = 1;

    public static function form(Form $form): Form
    {
        return $form->schema([
            Section::make('Order Details')->schema([
                Select::make('customer_id')
                    ->relationship('customer', 'name')
                    ->searchable()
                    ->preload()
                    ->required(),

                Select::make('status')
                    ->options(OrderStatus::class)   // use PHP enum
                    ->required(),

                TextInput::make('total_cents')
                    ->label('Total (cents)')
                    ->numeric()
                    ->required(),
            ])->columns(2),

            Section::make('Line Items')->schema([
                Repeater::make('items')
                    ->relationship()
                    ->schema([
                        Select::make('product_id')
                            ->relationship('product', 'name')
                            ->required(),
                        TextInput::make('quantity')->numeric()->required(),
                        TextInput::make('price_cents')->numeric()->required(),
                    ])
                    ->columns(3),
            ]),
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('id')->sortable(),
                TextColumn::make('customer.name')->searchable()->sortable(),
                BadgeColumn::make('status')
                    ->colors([
                        'warning' => OrderStatus::Placed,
                        'success' => OrderStatus::Shipped,
                        'danger'  => OrderStatus::Cancelled,
                    ]),
                TextColumn::make('total_cents')
                    ->label('Total')
                    ->formatStateUsing(fn ($state) => '$' . number_format($state / 100, 2))
                    ->sortable(),
                TextColumn::make('created_at')->dateTime()->sortable(),
            ])
            ->filters([
                SelectFilter::make('status')->options(OrderStatus::class),
                Filter::make('created_at')
                    ->form([
                        DatePicker::make('from'),
                        DatePicker::make('until'),
                    ])
                    ->query(fn (Builder $query, array $data) => $query
                        ->when($data['from'], fn ($q, $date) => $q->whereDate('created_at', '>=', $date))
                        ->when($data['until'], fn ($q, $date) => $q->whereDate('created_at', '<=', $date))
                    ),
            ])
            ->actions([Tables\Actions\EditAction::make()])
            ->bulkActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make(),
                    BulkAction::make('markShipped')
                        ->action(fn ($records) => $records->each->markShipped())
                        ->requiresConfirmation()
                        ->icon('heroicon-o-truck'),
                ]),
            ]);
    }

    public static function getRelations(): array
    {
        return [OrderItemsRelationManager::class];
    }

    public static function getPages(): array
    {
        return [
            'index'  => Pages\ListOrders::route('/'),
            'create' => Pages\CreateOrder::route('/create'),
            'edit'   => Pages\EditOrder::route('/{record}/edit'),
        ];
    }
}
```

## Relation Manager

```php
// app/Filament/Resources/OrderResource/RelationManagers/OrderItemsRelationManager.php
final class OrderItemsRelationManager extends RelationManager
{
    protected static string $relationship = 'items';

    public function form(Form $form): Form
    {
        return $form->schema([
            Select::make('product_id')->relationship('product', 'name')->required(),
            TextInput::make('quantity')->numeric()->required(),
        ]);
    }

    public function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('product.name'),
                TextColumn::make('quantity'),
            ])
            ->headerActions([Tables\Actions\CreateAction::make()])
            ->actions([Tables\Actions\EditAction::make(), Tables\Actions\DeleteAction::make()]);
    }
}
```

## Widgets

```php
// Stats overview widget
final class OrderStatsWidget extends BaseWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Total Orders', Order::count())
                ->description('All time')
                ->color('success'),
            Stat::make('Revenue', '$' . number_format(Order::sum('total_cents') / 100, 2))
                ->description('+12% this month')
                ->color('primary'),
            Stat::make('Pending', Order::where('status', 'placed')->count())
                ->color('warning'),
        ];
    }
}

// Chart widget
final class OrdersChart extends ChartWidget
{
    protected static ?string $heading = 'Orders per Day';

    protected function getData(): array
    {
        $data = Order::selectRaw('DATE(created_at) as date, COUNT(*) as count')
            ->groupBy('date')->orderBy('date')->get();

        return [
            'datasets' => [['label' => 'Orders', 'data' => $data->pluck('count')]],
            'labels'   => $data->pluck('date'),
        ];
    }

    protected function getType(): string { return 'line'; }
}
```

## Custom Pages

```php
php artisan make:filament-page Settings

// app/Filament/Pages/Settings.php
final class Settings extends Page
{
    protected static ?string $navigationIcon = 'heroicon-o-cog';
    protected static string $view = 'filament.pages.settings';

    public ?array $data = [];

    public function mount(): void
    {
        $this->form->fill(Setting::all()->pluck('value', 'key')->toArray());
    }

    public function save(): void
    {
        $data = $this->form->getState();
        foreach ($data as $key => $value) {
            Setting::updateOrCreate(['key' => $key], ['value' => $value]);
        }
        Notification::make()->title('Saved')->success()->send();
    }

    protected function getFormSchema(): array
    {
        return [
            TextInput::make('app_name')->required(),
            Toggle::make('maintenance_mode'),
        ];
    }
}
```

## Authorization

```php
// Gate-based resource authorization
final class OrderPolicy
{
    public function viewAny(User $user): bool   { return $user->hasRole('admin'); }
    public function create(User $user): bool    { return $user->hasRole('admin'); }
    public function update(User $user, Order $order): bool { return $user->hasRole('admin'); }
    public function delete(User $user, Order $order): bool { return $user->hasRole('super-admin'); }
}

// In Resource
public static function getEloquentQuery(): Builder
{
    // Automatically scoped to authorized records
    return parent::getEloquentQuery()->withoutGlobalScopes([SoftDeletingScope::class]);
}
```

## Multi-Tenancy with Filament

```php
// Panel provider
->tenant(Team::class)
->tenantRegistration(RegisterTeam::class)

// Model must implement HasTenants
class Order extends Model implements HasTenant
{
    use BelongsToTenant;  // scopes all queries to current team
}
```

## Common Patterns

**Soft deletes:**
```php
->softDeletes()  // adds trash filter + restore action automatically
```

**Global search:**
```php
protected static ?string $recordTitleAttribute = 'title';

public static function getGlobalSearchResultDetails(Model $record): array
{
    return ['Customer' => $record->customer->name];
}
```

**Notifications from actions:**
```php
Action::make('approve')
    ->action(function (Order $record) {
        $record->approve();
        Notification::make()->title('Order approved')->success()->send();
    })
    ->requiresConfirmation();
```
