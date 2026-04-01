---
name: ensi-openapi
description: Работайте с OpenAPI спецификациями в сервисах Ensi. Используйте этот скилл всегда, когда пользователь упоминает OpenAPI, API спецификации, создание endpoints, схем, перечислений или работу с yaml файлами в `public/api-docs/`. Также используйте при упоминании Swagger, спецификаций API, создании новых API endpoints или обновлении существующей документации API.
compatibility: "Requires access to PHP Laravel project with Ensi OpenAPI structure. Files should be in `public/api-docs/` directory with OpenAPI 3.0.1 format."
---

# Работа с OpenAPI спецификациями в сервисах Ensi

Этот скилл помогает работать с OpenAPI спецификациями в сервисах на базе Ensi Laravel OpenAPI Server Generator. Скилл обеспечивает понимание структуры спецификаций, создание новых endpoints, схем и перечислений, а также обновление конфигурации генерации.

## Структура OpenAPI спецификаций в Ensi

### Основная структура директорий

```
public/api-docs/v1/
├── index.yaml                    # Основной файл спецификации
├── common_parameters.yaml         # Общие параметры
├── common_schemas.yaml           # Общие схемы
├── errors.yaml                   # Ошибки
├── {module_name}/                # Модуль спецификации
│   ├── paths.yaml               # Пути модуля
│   ├── schemas/                 # Схемы модуля
│   │   └── {entity}.yaml        # Схема сущности
│   └── enums/                   # Перечисления модуля
│       └── {enum_name}.yaml     # Перечисление
```

### Основной файл спецификации (index.yaml)

Содержит:
- `openapi: 3.0.1` - версия спецификации
- `info` - информация о сервисе
- `servers` - URL сервера
- `tags` - теги для группировки endpoints
- `paths` - пути (референсы на модульные файлы)
- `components` - компоненты (схемы, параметры, ответы)

### Конфигурация генерации (config/openapi-server-generator.php)

Содержит маппинги для генерации кода:
- `api_docs_mappings` - маппинги OpenAPI файлов на пространства имен
- `namespaces_to_directories_mapping` - маппинги пространств имен на директории
- `supported_entities` - поддерживаемые сущности (controllers, enums, requests, routes, pest_tests, resources)
- `default_entities_to_generate` - сущности для генерации по умолчанию

## Создание новых API endpoints

### 1. Добавление нового модуля

При создании нового модуля:

1. Создайте директорию для модуля: `public/api-docs/v1/{module_name}/`
2. Создайте файл `paths.yaml` с описанием endpoints
3. Создайте директории `schemas/` и `enums/`
4. Добавьте тег в `index.yaml`
5. Добавьте референсы на пути в `index.yaml`

### 2. Добавление нового entity

Для новой сущности в существующем модуле:

1. Создайте схему в `public/api-docs/v1/{module_name}/schemas/{entity}.yaml`
2. Добавьте paths в `public/api-docs/v1/{module_name}/paths.yaml`
3. Добавьте референсы в `index.yaml`
4. При необходимости создайте enums

### 3. Структура schema файла

Схема сущности обычно содержит следующие компоненты:

```yaml
EntityReadonlyProperties:
  type: object
  properties:
    id:
      type: integer
      description: Идентификатор
    created_at:
      format: date-time
      type: string
    updated_at:
      format: date-time
      type: string

EntityFillableProperties:
  type: object
  properties:
    name:
      type: string
      description: Название

EntityRequiredFillableProperties:
  type: object
  required:
    - name

EntityIncludes:
  type: object
  properties:
    related_entity:
      type: array
      items:
        $ref: './related_entity.yaml#/RelatedEntity'

Entity:
  allOf:
    - $ref: '#/EntityReadonlyProperties'
    - $ref: '#/EntityFillableProperties'
    - $ref: '#/EntityRequiredFillableProperties'
    - $ref: '#/EntityIncludes'

CreateEntityRequest:
  allOf:
    - $ref: '#/EntityFillableProperties'
    - $ref: '#/EntityRequiredFillableProperties'

ReplaceEntityRequest:
  allOf:
    - $ref: '#/EntityFillableProperties'
    - $ref: '#/EntityRequiredFillableProperties'

PatchEntityRequest:
  type: object
  properties:
    name:
      type: string

SearchEntitiesFilter:
  type: object
  properties:
    name:
      type: string

SearchEntitiesRequest:
  type: object
  properties:
    sort:
      $ref: '../../common_schemas.yaml#/RequestBodySort'
    filter:
      $ref: '#/SearchEntitiesFilter'
    include:
      $ref: '../../common_schemas.yaml#/RequestBodyInclude'
    pagination:
      $ref: '../../common_schemas.yaml#/RequestBodyPagination'

EntityResponse:
  type: object
  properties:
    data:
      $ref: '#/Entity'
    meta:
      type: object
  required:
    - data

SearchEntitiesResponse:
  type: object
  properties:
    data:
      type: array
      items:
        $ref: '#/Entity'
    meta:
      type: object
      properties:
        pagination:
          $ref: '../../common_schemas.yaml#/ResponseBodyPagination'
  required:
    - data
    - meta
```

### 4. Структура paths файла

Пути описывают API endpoints и операции:

```yaml
Entities:
  post:
    tags:
      - entities
    operationId: createEntity
    x-lg-handler: 'App\Http\ApiV1\Modules\Entities\Controllers\EntitiesController@create'
    summary: Создание объекта типа Entity
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: './schemas/entities.yaml#/CreateEntityRequest'
    responses:
      "201":
        description: Успешный ответ
        content:
          application/json:
            schema:
              $ref: './schemas/entities.yaml#/EntityResponse'
      "400":
        $ref: '../index.yaml#/components/responses/BadRequest'
      "500":
        $ref: '../index.yaml#/components/responses/ServerError'

EntitiesSearch:
  post:
    tags:
      - entities
    operationId: searchEntities
    x-lg-handler: 'App\Http\ApiV1\Modules\Entities\Controllers\EntitiesController@search'
    x-lg-skip-request-generation: true
    summary: Поиск объектов типа Entity
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: './schemas/entities.yaml#/SearchEntitiesRequest'
    responses:
      "200":
        description: Успешный ответ
        content:
          application/json:
            schema:
              $ref: './schemas/entities.yaml#/SearchEntitiesResponse'
      "400":
        $ref: '../index.yaml#/components/responses/BadRequest'
      "500":
        $ref: '../index.yaml#/components/responses/ServerError'
```

## Создание перечислений (Enums)

### Структура enum файла

```yaml
EntityStatusEnum:
  type: integer
  enum:
    - 1
    - 2
    - 3
  description: Статус сущности
  x-enum-varnames:
    - ACTIVE
    - INACTIVE
    - DELETED
  x-enum-descriptions:
    - Активный
    - Неактивный
    - Удален
```

### Добавление enum в спецификацию

1. Создайте файл в `public/api-docs/v1/{module_name}/enums/{enum_name}.yaml`
2. Добавьте референс в `index.yaml` в `components.schemas`

```yaml
components:
  schemas:
    EntityStatusEnum:
      $ref: './{module_name}/enums/entity_status_enum.yaml'
```

## Обновление конфигурации генерации

При добавлении новых модулей или изменении структуры:

1. Проверьте `config/openapi-server-generator.php`
2. При необходимости добавьте новые маппинги в `api_docs_mappings`
3. Убедитесь что пространства имен правильные

## Правила и лучшие практики

### Именование
- Используйте snake_case для YAML файлов
- Используйте PascalCase для названий сущностей в YAML
- Используйте camelCase для operationId

### Референсы
- Всегда используйте `$ref` для повторного использования компонентов
- Относительные пути начинаются с `./` для текущей директории или `../` для родительской
- Формат: `$ref: '../../common_schemas.yaml#/SchemaName'`

### Генерация кода
- `x-lg-handler` определяет контроллер и метод для генерации
- `x-lg-skip-request-generation: true` пропускает генерацию request классов
- После изменения спецификации запустите генерацию

### Валидация
- Используйте Spectral для валидации: `spectral lint public/api-docs/v1/index.yaml`
- Проверьте что все референсы действительны

## Работа с существующей спецификацией

### Чтение спецификации
1. Начните с `public/api-docs/v1/index.yaml`
2. Изучите структуру модулей в директориях
3. Используйте Glob для поиска файлов по шаблону

### Добавление новых endpoints
1. Найдите существующие аналогичные endpoints
2. Скопируйте структуру и адаптируйте
3. Убедитесь что референсы правильные
4. Обновите `index.yaml`

### Обновление существующих схем
1. Найдите соответствующий YAML файл
2. Изучите существующие структуры
3. Добавьте новые поля или обновите существующие
4. Проверьте что референсы остаются актуальными

## Генерация кода из спецификации

После обновления спецификации:

```bash
# Генерация кода из OpenAPI спецификации
php artisan openapi:generate-server
```

Это создаст:
- Контроллеры в `app/Http/ApiV1/Modules/`
- Request классы
- Enum классы в `app/Http/ApiV1/OpenApiGenerated/Enums/`
- Routes в `app/Http/ApiV1/OpenApiGenerated/routes.php`
- Pest тесты

## Поиск и отладка

### Поиск компонентов
```bash
# Найти все YAML файлы спецификации
find public/api-docs/v1 -name "*.yaml"

# Найти конкретную сущность
grep -r "EntityName" public/api-docs/v1

# Проверить референсы
grep -r "EntityStatusEnum" public/api-docs/v1
```

### Типичные проблемы
- Сломанные референсы - проверьте пути файлов
- Дублирование названий - используйте уникальные имена
- Неправильные типы данных - проверьте соответствие типов
- Отсутствующие required поля - проверьте что все необходимые поля отмечены

## Примеры использования

### Пример 1: Создание нового простого entity
```
Пользователь: Создай API endpoint для управления отзывами пользователей
```
Создать:
- Модуль `reviews/`
- Схему `reviews.yaml` с базовыми CRUD операциями
- Перечисления если нужны
- Пути для CRUD операций
- Обновить `index.yaml`

### Пример 2: Добавление поля в существующую схему
```
Пользователь: Добавь поле 'priority' в схему Banner
```
Действия:
- Найти файл `public/api-docs/v1/contents/schemas/banners.yaml`
- Добавить поле в соответствующий компонент
- Проверить референсы

### Пример 3: Создание enum
```
Пользователь: Создай перечисление для статусов заказов
```
Действия:
- Создать файл `public/api-docs/v1/{module}/enums/order_status_enum.yaml`
- Добавить в `index.yaml`
- Определить значения и описания

## Шаблоны для копирования

### Минимальный template для нового entity
Скопируйте структуру из существующего entity, например `banners.yaml` или `pages.yaml`, и адаптируйте под новые требования.

### Минимальный template для нового paths
Скопируйте структуру из существующего paths файла, адаптируйте `operationId`, `x-lg-handler`, теги и референсы.

## Важно помнить

- Ensi использует OpenAPI 3.0.1
- Laravel OpenAPI Server Generator для генерации кода
- Структура спецификации влияет на сгенерированный код
- Всегда проверяйте валидность после изменений
- Используйте существующие компоненты для consistency
