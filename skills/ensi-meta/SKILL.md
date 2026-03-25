---
name: ensi-meta
description: Создание и модификация meta эндпоинтов в Ensi сервисах. Использовать при работе с мета-информацией полей, Field классами, ModelMetaResource, EnumInfo классами, а также при упоминании meta методов в контроллерах, создании мета-эндпоинтов, настройке фильтров/сортировки для фронтенда.
---

# Ensi Meta Endpoints

Meta эндпоинты возвращают структурированную информацию о полях сущности для фронтенда: типы полей, доступные фильтры, сортировку, отображение в списках.

## Расположение файлов

```
app/Http/ApiV1/Modules/{Module}/Controllers/{Entity}Controller.php  # meta() метод
app/Domain/Common/Data/Meta/Enum/{Domain}/{Enum}EnumInfo.php        # EnumInfo классы
app/Domain/Common/Data/Meta/Field.php                                # Фабрика полей
app/Domain/Common/Data/Meta/Fields/                                  # Классы полей
app/Http/ApiV1/Support/Resources/ModelMetaResource.php               # Ресурс ответа
```

## Ключевые классы

| Класс | Назначение |
|-------|------------|
| `Field` | Фабрика для создания полей разных типов |
| `ModelMetaResource` | Формирует JSON ответ с мета-информацией |
| `AbstractEnumInfo` | Базовый класс для информации о перечислениях |
| `AbstractField` | Базовый класс для всех типов полей |

## Типы полей (Field фабрика)

| Метод | Тип | Фильтр по умолчанию | Описание |
|-------|-----|---------------------|----------|
| `Field::id($code, $name)` | INT | Точный + sortDefault | Идентификатор |
| `Field::text($code, $name)` | STRING | LIKE | Текст с поиском по подстроке |
| `Field::string($code, $name)` | STRING | Нет | Простой текст (resetFilter) |
| `Field::keyword($code, $name)` | STRING | Точный | Ключевое слово |
| `Field::boolean($code, $name)` | BOOLEAN | Точный | Булево значение |
| `Field::enum($code, $name, $enumInfo)` | ENUM | MANY | Перечисление |
| `Field::datetime($code, $name)` | DATETIME | RANGE (gte/lte) | Дата и время |
| `Field::date($code, $name)` | DATE | RANGE (gte/lte) | Дата |
| `Field::integer($code, $name)` | INT | RANGE (gte/lte) | Целое число |
| `Field::float($code, $name)` | DOUBLE | RANGE (gte/lte) | Число с плавающей точкой |
| `Field::price($code, $name)` | PRICE | RANGE (gte/lte) | Цена |
| `Field::pluralNumeric($code, $name)` | PLURAL_NUMERIC | Нет | Число с несколькими типами |
| `Field::photo($code, $name)` | PHOTO | Нет | Фото |
| `Field::image($code, $name)` | IMAGE | Нет | Изображение |
| `Field::url($code, $name)` | URL | Нет | URL ссылка |
| `Field::phone($code, $name)` | PHONE | Нет | Телефон |
| `Field::email($code, $name)` | EMAIL | Нет | Email |

## Методы настройки поля

### Фильтрация

```php
Field::text('name', 'Название')
    ->filter()                    // Точное совпадение
    ->filter('custom_key')        // Точное совпадение с кастомным ключом
    ->filterLike()                // LIKE %value% (name_like)
    ->filterMany()                // Множественный фильтр (in)
    ->filterInclusiveRange()      // Диапазон gte/lte (name_gte, name_lte)
    ->filterExclusiveRange()      // Диапазон gt/lt (name_gt, name_lt)
    ->filterDefault()             // Показывать в фильтрах по умолчанию
    ->resetFilter()               // Сбросить настройки фильтра
```

### Сортировка

```php
Field::text('name', 'Название')
    ->sort()                      // Разрешить сортировку
    ->sort('custom_key')          // С кастомным ключом
    ->sortDefault()               // Сортировка по умолчанию (asc)
    ->sortDefault('id', 'desc')   // Сортировка по умолчанию (desc)
    ->resetSort()                 // Сбросить настройки сортировки
```

### Отображение в списке

```php
Field::text('name', 'Название')
    ->listDefault()               // Показывать в списке по умолчанию
    ->listHide()                  // Скрыть из списка
    ->detailLink()                // Поле как ссылка на детальную страницу (авто listDefault)
```

### Связи и вложенные объекты

```php
Field::text('author_name', 'Автор')
    ->object('author')            // Вложенный объект, загружается через include
    ->array()                     // Поле является массивом
    ->include('author')           // Ключ для загрузки связи
```

## EnumInfo классы

EnumInfo предоставляют информацию о возможных значениях перечислений для фильтров.

### Values-based (статические значения из enum класса)

```php
<?php

namespace App\Domain\Common\Data\Meta\Enum\Catalog;

use App\Domain\Common\Data\Meta\Enum\AbstractEnumInfo;
use App\Http\ApiV1\OpenApiGenerated\Enums\ProductTypeEnum;

class ProductTypeEnumInfo extends AbstractEnumInfo
{
    public function __construct()
    {
        $this->enumClassToValues(ProductTypeEnum::class);
    }
}
```

### Values-based (ручное добавление значений)

```php
<?php

namespace App\Domain\Common\Data\Meta\Enum\Custom;

use App\Domain\Common\Data\Meta\Enum\AbstractEnumInfo;

class StatusEnumInfo extends AbstractEnumInfo
{
    public function __construct()
    {
        $this->addValue(1, 'Активный')
             ->addValue(2, 'Неактивный')
             ->addValue(3, 'Удален');
    }
}
```

### Endpoint-based (ленивая загрузка через API)

```php
<?php

namespace App\Domain\Common\Data\Meta\Enum\Logistic;

use App\Domain\Common\Data\Meta\Enum\AbstractEnumInfo;

class RegionEnumInfo extends AbstractEnumInfo
{
    public function __construct()
    {
        $this->endpointName = 'logistic.searchRegionEnumValues';
        $this->endpointParams = []; // Опционально: параметры маршрута
    }
}
```

## Шаблон meta метода в контроллере

```php
use App\Domain\Common\Data\Meta\Field;
use App\Http\ApiV1\Support\Resources\ModelMetaResource;

public function meta(): ModelMetaResource
{
    return new ModelMetaResource([
        Field::id()->listDefault()->filterDefault()->detailLink(),
        Field::text('name', __('meta.name'))->sort()->filterDefault()->listDefault(),
        Field::text('code', __('meta.code'))->listDefault(),
        Field::boolean('is_active', __('meta.activity'))->listDefault()->filterDefault(),
        Field::integer('parent_id', __('meta.parent_id'))->filter(),
        Field::datetime('created_at', __('meta.created_at'))->resetFilter(),
        Field::datetime('updated_at', __('meta.updated_at'))->resetFilter(),
    ]);
}
```

## Meta метод с enum полем

```php
use App\Domain\Common\Data\Meta\Enum\Catalog\PropertyTypeEnumInfo;

public function meta(PropertyTypeEnumInfo $types): ModelMetaResource
{
    return new ModelMetaResource([
        Field::id()->listDefault()->filterDefault()->detailLink(),
        Field::text('name', __('meta.name'))->listDefault()->filterDefault(),
        Field::enum('type', __('meta.type'), $types)->listDefault()->filterDefault(),
        Field::boolean('is_active', __('meta.activity'))->listDefault()->filterDefault(),
        Field::datetime('created_at', __('meta.created_at'))->resetFilter(),
    ]);
}
```

## Полный пример контроллера

```php
<?php

namespace App\Http\ApiV1\Modules\Catalog\Controllers\Categories;

use App\Domain\Common\Data\Meta\Enum\Catalog\PropertyTypeEnumInfo;
use App\Domain\Common\Data\Meta\Field;
use App\Http\ApiV1\Modules\Catalog\Queries\Categories\CategoriesQuery;
use App\Http\ApiV1\Modules\Catalog\Resources\Categories\CategoriesResource;
use App\Http\ApiV1\Support\Resources\ModelMetaResource;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class CategoriesController
{
    public function search(CategoriesQuery $query): AnonymousResourceCollection
    {
        return CategoriesResource::collectPage($query->get());
    }

    public function get(int $id, CategoriesQuery $query): CategoriesResource
    {
        return new CategoriesResource($query->find($id));
    }

    public function meta(): ModelMetaResource
    {
        return new ModelMetaResource([
            Field::id()->listDefault()->filterDefault()->detailLink(),
            Field::text('name', __('meta.name'))->sort()->filterDefault()->listDefault(),
            Field::text('code', __('meta.code'))->listDefault(),
            Field::integer('parent_id', __('meta.categories.parent_id'))->filter(),
            Field::boolean('is_active', __('meta.categories.is_active'))->listDefault(),
            Field::datetime('created_at', __('meta.created_at'))->resetFilter(),
            Field::datetime('updated_at', __('meta.updated_at'))->resetFilter(),
        ]);
    }
}
```

## Best Practices

1. **Цепочки методов** - используйте fluent interface для компактной настройки
2. **Локализация** - используйте `__('meta.field_name')` для названий полей
3. **ID поле** - всегда включайте `Field::id()` с `detailLink()`
4. **Timestamps** - включайте `created_at`, `updated_at` с `resetFilter()`
5. **EnumInfo** - создавайте в `Domain/Common/Data/Meta/Enum/{Domain}/`
6. **Dependency Injection** - EnumInfo классы внедряйте через параметры метода
7. **Фильтры по умолчанию** - используйте `filterDefault()` для часто используемых фильтров
8. **Скрытые поля** - используйте `listHide()` + `filter()` для полей только для фильтрации

## Типичные ошибки

1. **Вызов filterDefault() без filter()** - выбросит исключение
2. **Вызов sort() на listHide() поле** - выбросит исключение
3. **EnumInfo без значений** - выбросит исключение при сериализации
4. **Отсутствие метода getDescriptions()** - при использовании enumClassToValues()
