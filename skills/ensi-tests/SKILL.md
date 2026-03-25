---
name: ensi-tests
description: Work with Pest PHP tests in Ensi services. Use this skill whenever the user mentions creating tests, writing tests, testing API endpoints, testing components, unit testing, or test files in Ensi Laravel projects. This includes requests like "create a test for this endpoint", "write component tests", "add unit tests", "create test coverage", or any mention of Pest PHP, PHPUnit, or testing in the context of Ensi services.
---

# Ensi Tests Skill

This skill helps you create and work with Pest PHP tests in Ensi Laravel services. Ensi uses Pest v2.0 with Laravel plugin and follows specific testing patterns.

## Understanding Ensi Testing Structure

### Test Types

Ensi projects use three main test types:

1. **Component Tests** (`*ComponentTest.php`) - Test API endpoints and HTTP layer
2. **Integration Tests** (`*IntegrationTest.php`) - Test database and external service integration
3. **Unit Tests** (`*UnitTest.php`) - Test isolated classes and methods

### Base Classes

All tests extend from base classes:

- `Tests\TestCase` - Base class for all tests (uses WithFakerProviderTestCase)
- `Tests\ComponentTestCase` - For component tests (uses DatabaseTransactions, MockServicesApi)
- `Tests\IntegrationTestCase` - For integration tests (uses DatabaseTransactions, MockServicesApi)
- `Tests\UnitTestCase` - For unit tests (has mockQueryBuilder helper)
- `App\Http\ApiV1\Support\Tests\ApiV1ComponentTestCase` - For API v1 component tests (validates against OpenAPI spec)

### Test File Location

Tests are organized by module:
```
app/Http/ApiV1/Modules/{ModuleName}/Tests/{TestName}ComponentTest.php
app/Http/ApiV1/Modules/{ModuleName}/Tests/Factories/{Entity}RequestFactory.php
app/Domain/{DomainName}/Tests/{TestName}UnitTest.php
```

## Creating Component Tests

Component tests test API endpoints. Follow this pattern:

### Basic Structure

```php
<?php

use App\Domain\{Domain}\Models\{Model};
use App\Http\ApiV1\Support\Tests\ApiV1ComponentTestCase;
use function Pest\Laravel\postJson;

uses(ApiV1ComponentTestCase::class);
uses()->group('component');

test('POST /api/v1/{resource}:search 200', function () {
    $models = {Model}::factory()->count(5)->create();

    postJson('/api/v1/{resource}:search', [
        "sort" => ["-id"],
    ])
        ->assertStatus(200)
        ->assertJsonCount(5, 'data')
        ->assertJsonPath('data.0.id', $models->last()->id);
});
```

### Test Naming Convention

Always use this format: `'{HTTP_METHOD} /{endpoint} {status_code}'`

Examples:
- `'POST /api/v1/contents/banners:search 200'`
- `'POST /api/v1/contents/banners 201'` (for create operations)
- `'PUT /api/v1/contents/banners/{id} 200'`
- `'DELETE /api/v1/contents/banners/{id} 200'`

### Common Test Patterns

#### Search with Filters

```php
test("POST /api/v1/{resource}:search filter success", function (string $fieldKey, $value = null, ?string $filterKey = null, $filterValue = null) {
    $model = {Model}::factory()->create($value ? [$fieldKey => $value] : []);
    {Model}::factory()->create();

    postJson("/api/v1/{resource}:search", ["filter" => [
        ($filterKey ?: $fieldKey) => ($filterValue ?: $model->{$fieldKey}),
    ], 'sort' => ['id'], 'pagination' => ['type' => PaginationTypeEnum::OFFSET, 'limit' => 1]])
        ->assertStatus(200)
        ->assertJsonCount(1, 'data')
        ->assertJsonPath('data.0.id', $model->id);
})->with([
    ['name', 'test name'],
    ['is_active', true],
    ['name', 'test', 'name_like', 'te'],
]);
```

#### Create Operation

```php
test('POST /api/v1/{resource} 200', function () {
    $request = {RequestFactory}::new()->make();

    postJson('/api/v1/{resource}', $request)
        ->assertStatus(201);

    assertDatabaseHas((new {Model}())->getTable(), [
        'name' => $request['name'],
    ]);
});
```

#### Update Operation

```php
test('PUT /api/v1/{resource}/{id} 200', function () {
    $model = {Model}::factory()->create();
    $request = {RequestFactory}::new()->make();

    putJson("/api/v1/{resource}/{$model->id}", $request)
        ->assertJsonPath('data.name', $request['name'])
        ->assertStatus(200);

    assertDatabaseHas((new {Model}())->getTable(), [
        'id' => $model->id,
        'name' => $request['name'],
    ]);
});
```

#### Delete Operation

```php
test('DELETE /api/v1/{resource}/{id} 200', function () {
    $model = {Model}::factory()->create();

    deleteJson("/api/v1/{resource}/{$model->id}")
        ->assertStatus(200);

    assertModelMissing($model);
});
```

#### 404 Tests

```php
test('GET /api/v1/{resource}/{id} 404', function () {
    getJson("/api/v1/{resource}/999")
        ->assertStatus(404);
});
```

### Common Assertions

Use these Pest assertions for API tests:

- `->assertStatus($code)` - Check HTTP status code
- `->assertJsonPath('path', $value)` - Check JSON path value
- `->assertJsonStructure(['data' => ['field1', 'field2']])` - Check JSON structure
- `->assertJsonCount($count, 'data')` - Check array count
- `->assertCreated()` - Alias for 201 status
- `->assertOk()` - Alias for 200 status

### Database Assertions

```php
use function Pest\Laravel\assertDatabaseHas;
use function Pest\Laravel\assertDatabaseMissing;
use function Pest\Laravel\assertModelMissing;

assertDatabaseHas((new Model())->getTable(), ['field' => 'value']);
assertDatabaseMissing((new Model())->getTable(), ['field' => 'value']);
assertModelMissing($model); // Model no longer exists
```

## Creating Unit Tests

Unit tests test isolated classes and methods:

### Basic Structure

```php
<?php

use Tests\UnitTestCase;

uses(UnitTestCase::class);

test('method_name does something', function () {
    $class = new ClassName();
    $result = $class->method();

    expect($result)->toBe('expected value');
});
```

### Using Mocked Query Builder

```php
test('query builder method works correctly', function () {
    $builder = $this->mockQueryBuilder();

    $builder->shouldReceive('where')
        ->once()
        ->with('field', 'value')
        ->andReturnSelf();

    // Test your code that uses the query builder
});
```

### Expectation Assertions

Pest provides expectation-based assertions:

```php
expect($value)->toBe($expected);
expect($value)->toBeTrue();
expect($value)->toBeFalse();
expect($value)->toBeInstanceOf(ClassName::class);
expect($array)->toHaveCount(3);
expect($array)->toHaveKey('key');
expect($array)->sequence(...)->each->...
```

## Creating Request Factories

Request Factories generate test data for API requests. They inherit from `BaseApiFactory`:

### Basic Factory Structure

```php
<?php

namespace App\Http\ApiV1\Modules\{Module}\Tests\Factories;

use Ensi\LaravelTestFactories\BaseApiFactory;

class {Entity}RequestFactory extends BaseApiFactory
{
    protected function definition(): array
    {
        return [
            'name' => $this->faker->words(3, true),
            'code' => $this->faker->unique()->slug(),
            'is_active' => $this->faker->boolean(),
        ];
    }

    public function make(array $extra = []): array
    {
        return $this->makeArray($extra);
    }
}
```

### Factory with Modifiers

```php
class BannerRequestFactory extends BaseApiFactory
{
    protected bool $withButton = false;
    protected bool $useSort = false;

    protected function definition(): array
    {
        $data = [
            'name' => $this->faker->words(3, true),
            'is_active' => $this->faker->boolean,
        ];

        if ($this->withButton) {
            $data['button'] = [
                'url' => $this->faker->url,
                'text' => $this->faker->sentence,
            ];
        }

        if ($this->useSort) {
            $data['sort'] = $this->faker->numberBetween();
        }

        return $data;
    }

    public function withButton(bool $withButton = true): self
    {
        $this->withButton = $withButton;
        return $this;
    }

    public function withSort(bool $useSort = true): self
    {
        $this->useSort = $useSort;
        return $this;
    }

    public function make(array $extra = []): array
    {
        return $this->makeArray($extra);
    }
}
```

### Using Enums in Factories

```php
use App\Http\ApiV1\OpenApiGenerated\Enums\SomeEnum;

protected function definition(): array
{
    return [
        'type' => $this->faker->randomElement(SomeEnum::cases())->value,
        'location' => $this->faker->randomElement(AnotherEnum::cases())->value,
    ];
}
```

### Factory Location

Place factories in:
```
app/Http/ApiV1/Modules/{ModuleName}/Tests/Factories/
```

## Workflow for Creating Tests

When a user asks to create tests:

1. **Understand the context**: What module? What resource? What operations need testing?

2. **Check for existing patterns**: Look at existing tests in the same module to maintain consistency:
   - Read existing test files in `app/Http/ApiV1/Modules/{ModuleName}/Tests/`
   - Note the patterns used for similar operations

3. **Identify what's needed**:
   - Component tests for API endpoints?
   - Unit tests for service classes?
   - Request factories for test data?

4. **Ask about Request Factories if needed**: If test data generation would benefit from a factory, ask the user if they want one created.

5. **Create test files**: Follow the naming conventions and patterns above.

6. **Ensure proper imports**: Include all necessary Pest functions and model imports.

## Test Organization Tips

- Use `->with([...])` for data-driven tests with multiple scenarios
- Group related tests with `uses()->group('component', 'module-name')`
- Keep tests focused - one assertion or one logical flow per test
- Use descriptive test names that explain what's being tested
- Test both happy paths and edge cases (400, 404, validation errors)

## Common Pest Laravel Functions

Import these at the top of test files:

```php
use function Pest\Laravel\getJson;
use function Pest\Laravel\postJson;
use function Pest\Laravel\putJson;
use function Pest\Laravel\patchJson;
use function Pest\Laravel\deleteJson;
use function Pest\Laravel\assertDatabaseHas;
use function Pest\Laravel\assertDatabaseMissing;
use function Pest\Laravel\assertModelMissing;
```

## Example: Complete Test File

Here's a complete example of a component test file:

```php
<?php

use App\Domain\Contents\Models\Page;
use App\Http\ApiV1\Modules\Pages\Tests\Factories\PageRequestFactory;
use App\Http\ApiV1\Support\Tests\ApiV1ComponentTestCase;

use function Pest\Laravel\assertDatabaseHas;
use function Pest\Laravel\deleteJson;
use function Pest\Laravel\getJson;
use function Pest\Laravel\postJson;

uses(ApiV1ComponentTestCase::class);
uses()->group('component');

test('POST /api/v1/contents/pages:search 200', function () {
    Page::factory()->count(5)->create();

    $latest = Page::query()->latest('name')->first();

    postJson('/api/v1/contents/pages:search', ['sort' => ['-name']])
        ->assertStatus(200)
        ->assertJsonPath('data.0.id', $latest->id);
});

test('POST /api/v1/contents/pages 200 create page', function () {
    $pageData = PageRequestFactory::new()->make();

    postJson('/api/v1/contents/pages', $pageData)
        ->assertCreated()
        ->assertJsonPath('data.name', $pageData['name']);

    assertDatabaseHas((new Page())->getTable(), [
        'name' => $pageData['name'],
    ]);
});

test('DELETE /api/v1/contents/pages/{id} 200', function () {
    $page = Page::factory()->create();

    deleteJson("/api/v1/contents/pages/{$page->id}")
        ->assertStatus(200);

    assertDatabaseMissing((new Page())->getTable(), [
        'id' => $page->id,
    ]);
});
```

## Summary

- Use Pest PHP v2.0 syntax with `test()` function
- Follow naming convention: `{METHOD} /{endpoint} {status}`
- Use appropriate base class for test type
- Create Request Factories for complex test data
- Use Pest's expectation-based assertions for unit tests
- Use Laravel's HTTP assertions for component tests
- Maintain consistency with existing tests in the module
