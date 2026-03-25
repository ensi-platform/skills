---
name: ensi-query-builder
description: Создание и модификация Query классов для spatie/laravel-query-builder в Ensi сервисах. Использовать при работе с Query классами, фильтрами API, search endpoints, allowedFilters, allowedSorts, allowedIncludes, а также при упоминании фильтрации в контроллерах API, создании search-эндпоинтов, добавлении фильтров к моделям.
---

# Ensi Query Builder

Query классы обеспечивают фильтрацию, сортировку и include связей для API endpoints в Ensi сервисах.

## Расположение

```
app/Http/ApiV1/Modules/{Module}/Queries/{Entity}Query.php
```

## Базовый шаблон

```php
<?php

namespace App\Http\ApiV1\Modules\{Module}\Queries;

use App\Domain\{Domain}\Models\{Model};
use Ensi\QueryBuilderHelpers\Filters\NumericFilter;
use Spatie\QueryBuilder\QueryBuilder;

class {Entity}Query extends QueryBuilder
{
    public function __construct()
    {
        parent::__construct({Model}::query());

        $this->allowedIncludes(['relation1', 'relation2']);

        $this->allowedSorts(['id', 'name', 'created_at']);

        $this->allowedFilters([
            ...NumericFilter::make('id')->exact(),
        ]);

        $this->defaultSort('id');
    }
}
```

## Фильтры

### Spatie базовые фильтры

```php
use Spatie\QueryBuilder\AllowedFilter;

AllowedFilter::exact('status')              // Точное совпадение
AllowedFilter::partial('name')              // LIKE %value%
AllowedFilter::scope('active', 'scopeActive') // Local scope модели
AllowedFilter::trashed()                    // Soft deletes: with/only
AllowedFilter::belongsTo('category')        // Фильтр по BelongsTo связи
AllowedFilter::callback('custom', fn($q, $v) => $q->where('field', $v))
```

### Ensi Helpers (основной способ)

Пакет `ensi/laravel-query-builder-helpers` предоставляет типизированные фильтры с автоматическими суффиксами.

#### StringFilter

```php
use Ensi\QueryBuilderHelpers\Filters\StringFilter;

...StringFilter::make('name')
    ->exact()        // name
    ->contain()      // name_like
    ->startWith()    // name_llike
    ->endWith()      // name_rlike
    ->empty()        // name_empty
    ->not()          // name_not
```

#### NumericFilter

```php
use Ensi\QueryBuilderHelpers\Filters\NumericFilter;

...NumericFilter::make('price')
    ->exact()        // price
    ->gt()           // price_gt
    ->gte()          // price_gte
    ->lt()           // price_lt
    ->lte()          // price_lte
    ->empty()        // price_empty
    ->not()          // price_not
```

#### DateFilter / DateTimeFilter

```php
use Ensi\QueryBuilderHelpers\Filters\DateFilter;
use Ensi\QueryBuilderHelpers\Filters\DateTimeFilter;

...DateFilter::make('created_at')
    ->exact()        // created_at (дата)
    ->gte()          // created_at_gte
    ->lte()          // created_at_lte
    ->empty()        // created_at_empty
    ->not()          // created_at_not

// DateTimeFilter - аналогично, но для datetime полей
...DateTimeFilter::make('updated_at')->gte()->lte()
```

#### ArrayFilter

```php
use Ensi\QueryBuilderHelpers\Filters\ArrayFilter;

...ArrayFilter::make('tags')
    ->exact()        // tags - точное совпадение массива
    ->contain()      // tags_like - содержит значение
```

#### ExtraFilter

Для JSON полей и сложных случаев:

```php
use Ensi\QueryBuilderHelpers\Filters\ExtraFilter;

// JSON поле
ExtraFilter::contain('address_string', 'address->address_string')

// Предопределенный фильтр (выполняется если передано true)
ExtraFilter::predefined('only_active', fn($q) => $q->where('active', true))

// Вложенные фильтры по связи
...ExtraFilter::nested('items', [
    ExtraFilter::greater('item_price'),
])

// Дополнительные методы
ExtraFilter::has('items')      // Проверка наличия связи
ExtraFilter::empty('field')    // null или пустота
ExtraFilter::not('field')      // Отрицание
```

### Суффиксы фильтров

Суффиксы автоматически добавляются к имени фильтра:

| Суффикс | Описание | Пример |
|---------|----------|--------|
| (без суффикса) | Равенство | `id=1` |
| `_gt` | Больше | `price_gt=100` |
| `_gte` | Больше или равно | `created_at_gte=2024-01-01` |
| `_lt` | Меньше | `price_lt=1000` |
| `_lte` | Меньше или равно | `created_at_lte=2024-12-31` |
| `_like` | Содержит (LIKE %value%) | `name_like=ivan` |
| `_llike` | Начинается с (LIKE %value) | `code_llike=ABC` |
| `_rlike` | Заканчивается на (LIKE value%) | `email_rlike=@gmail.com` |
| `_not` | Не равно | `status_not=draft` |
| `_empty` | Пустое/null | `deleted_at_empty=true` |

### Комбинирование фильтров

```php
$this->allowedFilters([
    // Один фильтр
    AllowedFilter::exact('status'),
    
    // Множественные фильтры через spread
    ...NumericFilter::make('id')->exact(),
    ...StringFilter::make('name')->exact()->contain(),
    ...DateFilter::make('created_at')->gte()->lte(),
]);
```

### Фильтры по связям

```php
// Через dot-notation (автоматический whereHas)
...StringFilter::make('contact_name', 'contacts.name')->exact()

// Через callback для сложной логики
AllowedFilter::callback('seller_id', function (Builder $query, $value) {
    $query->whereHas('store', fn($q) => $q->where('seller_id', $value));
})
```

## Includes

```php
use Spatie\QueryBuilder\AllowedInclude;

// Простые связи
$this->allowedIncludes(['posts', 'comments', 'author']);

// С callback для сортировки/фильтрации связи
$this->allowedIncludes([
    AllowedInclude::callback('workings', function ($query) {
        return $query->orderBy('id', 'desc');
    }),
    AllowedInclude::callback('contacts', fn($q) => $q->orderBy('id', 'desc')),
]);

// С алиасом (третий параметр - реальное имя связи)
AllowedInclude::callback('pickup_times', fn($q) => $q->orderBy('id'), 'pickupTimes')
```

## Sorts

```php
// Разрешенные поля для сортировки
$this->allowedSorts([
    'id',
    'name',
    'status',
    'created_at',
    'updated_at',
]);

// Сортировка по умолчанию
$this->defaultSort('id');

// Обратная сортировка по умолчанию
$this->defaultSort('-created_at');
```

## Полный пример

```php
<?php

namespace App\Http\ApiV1\Modules\Stores\Queries;

use App\Domain\Stores\Models\Store;
use Ensi\QueryBuilderHelpers\Filters\DateFilter;
use Ensi\QueryBuilderHelpers\Filters\ExtraFilter;
use Ensi\QueryBuilderHelpers\Filters\NumericFilter;
use Ensi\QueryBuilderHelpers\Filters\StringFilter;
use Spatie\QueryBuilder\AllowedInclude;
use Spatie\QueryBuilder\QueryBuilder;

class StoresQuery extends QueryBuilder
{
    public function __construct()
    {
        parent::__construct(Store::query());

        $this->allowedIncludes([
            AllowedInclude::callback('workings', fn($q) => $q->orderBy('id', 'desc')),
            AllowedInclude::callback('contacts', fn($q) => $q->orderBy('id', 'desc')),
            'contact',
        ]);

        $this->allowedSorts([
            'id',
            'seller_id',
            'name',
            'active',
            'created_at',
            'updated_at',
        ]);

        $this->allowedFilters([
            ...NumericFilter::make('id')->exact(),
            ...NumericFilter::make('seller_id')->exact(),
            ...NumericFilter::make('active')->exact(),
            ...StringFilter::make('name')->exact()->contain(),
            ...StringFilter::make('xml_id')->exact(),
            ExtraFilter::contain('address_string', 'address->address_string'),
            // Фильтр по связи
            ...StringFilter::make('contact_name', 'contacts.name')->exact(),
            ...DateFilter::make('created_at')->gte()->lte(),
            ...DateFilter::make('updated_at')->gte()->lte(),
        ]);

        $this->defaultSort('id');
    }
}
```

## Best Practices

1. **Использовать Ensi helpers** - `NumericFilter`, `StringFilter`, `DateFilter` вместо чистого Spatie для стандартных фильтров
2. **Spread operator (`...`)** - обязательно при использовании Ensi helpers, так как методы возвращают массивы
3. **Группировать по типу** - все фильтры одной сущности через chain methods
4. **Фильтры по связям** - использовать dot-notation или callback для сложных случаев
5. **Default sort** - всегда указывать сортировку по умолчанию (обычно `id`)
6. **JSON поля** - через `ExtraFilter::contain()` с указанием JSON path
7. **Callback фильтры** - для логики, которую нельзя выразить через helpers
