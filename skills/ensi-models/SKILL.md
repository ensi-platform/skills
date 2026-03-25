---
name: ensi-models
description: Work with Eloquent models in Ensi projects following project standards. Use this skill whenever the user mentions creating, modifying, or working with Laravel Eloquent models, database models, model factories, model relationships, model scopes, database migrations, or database tables in the Ensi context. This includes requests like "create a new model", "add a relationship to this model", "create a factory for this model", "add a scope method", "create a migration", or any work related to database modeling and Laravel Eloquent patterns in Ensi projects.
---

# Ensi Models Skill

This skill helps you work with Eloquent models in Ensi projects following the project's established patterns and conventions.

## Project Structure Understanding

Ensi uses Domain-Driven Design (DDD) with the following structure:

- **Models**: `app/Domain/{DomainName}/Models/`
- **Factories**: `app/Domain/{DomainName}/Models/Tests/Factories/`
- All models extend `Illuminate\Database\Eloquent\Model`
- All factories extend `Ensi\LaravelTestFactories\BaseModelFactory`

## Key Patterns and Conventions

### Model Patterns

1. **PHPDoc Annotations**: Use PHPDoc for all properties with Russian descriptions
2. **Date Types**: Use `CarbonInterface` for date fields
3. **Type Safety**: Use fully typed relationships and methods
4. **Factory Method**: Every model should have a `factory()` method
5. **Scopes**: Use scopes for complex query logic

### Factory Patterns

1. **Base Extension**: Extend `BaseModelFactory` from Ensi
2. **Definition Method**: Implement `definition()` with faker data
3. **State Methods**: Create descriptive state methods (e.g., `active()`, `inactive()`)

### Common Properties

Standard model properties include:
- `$table` - database table name
- `$fillable` - mass assignable fields
- `$casts` - type casting rules
- `$attributes` - default attribute values
- Constants for default values (e.g., `DEFAULT_SORT`)

## Workflow

When working with models in Ensi service:

### 1. Understand the Context
- Identify the domain (e.g., Contents, Seo, Nameplates)
- Check for existing models in the same domain
- Look at existing patterns in similar models

### 2. Create Models
- Follow PHPDoc patterns with Russian descriptions
- Include proper type hints
- Add relationships with return type hints
- Include factory() method
- Use appropriate casts for data types

### 3. Create Factories
- Extend BaseModelFactory
- Use $this->faker for test data
- Create meaningful state methods
- Handle relationships properly

### 4. Add Relationships
- Use proper relationship types (BelongsTo, HasMany, etc.)
- Type hint return values
- Include PHPDoc for related properties
- Follow naming conventions

### 5. Add Scopes
- Prefix with `scope`
- Accept Builder as first parameter
- Return Builder instance
- Use descriptive names

### 6. Create Migrations
- Follow Laravel migration patterns
- Use appropriate column types
- Include indexes and constraints
- Follow naming conventions

## Code Examples

### Model Example

```php
<?php

namespace App\Domain\Contents\Models;

use App\Domain\Contents\Models\Tests\Factories\BannerFactory;
use Carbon\CarbonInterface;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

/**
 * @property int $id - id баннера
 * @property string $name - имя
 * @property bool $is_active - активность баннера
 * @property CarbonInterface $created_at
 * @property CarbonInterface $updated_at
 *
 * @property-read BannerType|null $type
 */
class Banner extends Model
{
    protected $table = 'banners';

    protected $fillable = [
        'name',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'bool',
    ];

    public function type(): BelongsTo
    {
        return $this->belongsTo(BannerType::class);
    }

    public static function factory(): BannerFactory
    {
        return BannerFactory::new();
    }
}
```

### Factory Example

```php
<?php

namespace App\Domain\Contents\Models\Tests\Factories;

use App\Domain\Contents\Models\Banner;
use Ensi\LaravelTestFactories\BaseModelFactory;

class BannerFactory extends BaseModelFactory
{
    protected $model = Banner::class;

    public function definition(): array
    {
        return [
            'name' => $this->faker->words(3, true),
            'is_active' => $this->faker->boolean,
        ];
    }

    public function active(): static
    {
        return $this->state([
            'is_active' => true,
        ]);
    }
}
```

## Best Practices

1. **Consistency**: Follow existing patterns in the project
2. **Type Safety**: Always use type hints and return types
3. **Documentation**: Include clear PHPDoc comments
4. **Testing**: Always create factories with models
5. **Relationships**: Use proper relationship types and foreign key conventions
6. **Scopes**: Create reusable query logic in scopes
7. **Naming**: Use clear, descriptive names following conventions

## Common Tasks

- **Create new model**: Generate model with factory following domain structure
- **Add relationship**: Add typed relationship method with PHPDoc
- **Create factory**: Generate factory with meaningful state methods
- **Add scope**: Create scope method for common queries
- **Create migration**: Generate migration following Laravel conventions
- **Modify existing model**: Update while maintaining patterns and consistency

Always examine existing models in the same domain to maintain consistency across the codebase.
