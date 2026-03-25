---
name: ensi-code-style
description: Enforce PHP and Laravel code style according to Ensi guidelines. Use this skill whenever writing, modifying, or reviewing PHP/Laravel code in Ensi projects, creating new classes (Controllers, Models, Actions, Events, etc.), refactoring existing code, setting up validation, routes, migrations, or working with Domain/Http layers in Laravel applications. Also trigger when checking code style compliance or generating Laravel components to ensure they follow Ensi conventions.
---

# Ensi Code Style Guide

This skill ensures that PHP and Laravel code follows the official Ensi code style guidelines. Apply these rules consistently when writing, reviewing, or refactoring code in Ensi projects.

## Core Principles

**Always follow these priorities:**
1. Existing codebase conventions override these guidelines
2. Code should be readable and maintainable
3. Follow PSR-1, PSR-2, and PSR-12
4. Use type hints and modern PHP features
5. Favor early returns over deep nesting
6. Minimize comments by writing self-documenting code

## PHP Style Rules

### Return Types and Properties

**Use void return type for methods that don't return anything:**
```php
// Good
public function scopeArchived(Builder $query): void
{
    $query->where(/* ... */);
}

// Bad
public function scopeArchived(Builder $query)
{
    $query->where(/* ... */);
}
```

**Use typed properties instead of PHPDoc:**
```php
// Good
class Foo
{
    public string $bar;
}

// Bad
class Foo
{
    /** @var string */
    public $bar;
}
```

### Constructor Property Promotion

**Promote properties in constructor when possible:**
```php
// Good - one property per line, comma after last
class MyClass
{
    public function __construct(
        protected string $firstArgument,
        protected string $secondArgument,
    ) {}
}

// Bad
class MyClass
{
    protected string $secondArgument;

    public function __construct(protected string $firstArgument, string $secondArgument)
    {
        $this->secondArgument = $secondArgument;
    }
}
```

### Documentation Blocks

**Only use PHPDoc when it adds context beyond type hints:**
```php
// Good - no PHPDoc needed, types are clear
class Url
{
    public static function fromString(string $url): Url
    {
        // ...
    }
}

// Bad - redundant PHPDoc
class Url
{
    /**
     * Create a url from a string.
     *
     * @param string $url
     *
     * @return \Foo\Url\Url
     */
    public static function fromString(string $url): Url
    {
        // ...
    }
}
```

**Use single-line PHPDoc for simple annotations:**
```php
/** @var string */
/** @test */
```

### Traits

**One trait per use statement:**
```php
// Good
class MyClass
{
    use TraitA;
    use TraitB;
}

// Bad
class MyClass
{
    use TraitA, TraitB;
}
```

### String Handling

**Prefer string interpolation over concatenation:**
```php
// Good
$greeting = "Hi, I am {$name}.";

// Bad
$greeting = 'Hi, I am ' . $name . '.';
```

### Ternary Operators

**Keep short ternaries on one line, long ones on separate lines:**
```php
// Good - short
$name = $isFoo ? 'foo' : 'bar';

// Good - multi-line
$result = $object instanceof Model
    ? $object->name
    : 'A default value';
```

### If Statements

**Always use curly braces:**
```php
// Good
if ($condition) {
    // code
}

// Bad
if ($condition)
    // code
```

**Handle failure paths first with early returns:**
```php
// Good
if (!$goodCondition) {
    throw new Exception;
}

// do work

// Bad
if ($goodCondition) {
    // do work
}

throw new Exception;
```

**Avoid else by using early returns:**
```php
// Good
if (!$conditionA) {
    return;
}

if (!$conditionB) {
    return;
}

if (!$conditionC) {
    return;
}

// conditions A, B, and C are met

// Bad
if ($conditionA) {
    if ($conditionB) {
        // conditions A and B met
    } else {
        // condition A met, B not
    }
} else {
    // condition A not met
}
```

**Prefer separate if statements over compound conditions:**
```php
// Good
if (!$conditionA) {
    return;
}

if (!$conditionB) {
    return;
}

if (!$conditionC) {
    return;
}

// do work

// Bad
if ($conditionA && $conditionB && $conditionC) {
    // do work
}
```

### Comments

**Minimize comments by writing self-documenting code:**
```php
// Good - function name replaces comment
function calculateCreditRating() { /* ... */ }

// Bad
// Starts credit rating calculation
function calc() { /* ... */ }
```

**Format comments with space after slashes:**
```php
// Good
// Comment with space

/* Good
 * Multi-line comment
 * with proper formatting
 */
```

### Whitespace

**Add blank lines between operators:**
```php
// Good
public function getPage($url)
{
    $page = $this->pages()->where('slug', $url)->first();

    (!$page) {
        return null;
    }

    if ($page['private'] && !Auth::check()) {
        return null;
    }

    return $page;
}
```

**No extra blank lines in braces:**
```php
// Good
if ($foo) {
    $this->foo = $foo;
}

// Bad
if ($foo) {

    $this->foo = $foo;

}
```

## Laravel Style Rules

### Configuration

**Config files use kebab-case:**
```
config/
  pdf-generator.php
```

**Config keys use snake_case:**
```php
// config/pdf-generator.php
return [
    'chrome_path' => env('CHROME_PATH'),
];
```

**Never call env() outside config files - use config() instead.**

### Artisan Commands

**Command names use kebab-case:**
```php
// Good
php artisan delete-old-records

// Bad
php artisan deleteOldRecords
```

### Routing

**Public URLs use kebab-case:**
```
https://project.ru/admin/product-ratings
```

**Use array syntax for controllers:**
```php
// Good
Route::get('product-ratings', [ProductRatingsController::class, 'index']);

// Bad
Route::get('product-ratings', 'ProductRatingsController@index');
```

**Route names use camelCase:**
```php
// Good
Route::get('product-ratings', [ProductRatingsController::class, 'index'])->name('productRatings');

// Bad
Route::get('product-ratings', [ProductRatingsController::class, 'index'])->name('product-ratings');
```

**Specify HTTP method first:**
```php
// Good
Route::get('/', [HomeController::class, 'index'])->name('home');
Route::get('product-ratings', [ProductRatingsController::class, 'index'])->name('productRatings');

// Bad
Route::name('home')->get('/', [HomeController::class, 'index']);
Route::name('openSource')->get('product-ratings', [ProductRatingsController::class, 'index']);
```

**Route parameters use camelCase:**
```php
Route::get('news/{newsItem}', [NewsItemsController::class, 'index']);
```

**URLs don't start with / except root:**
```php
// Good
Route::get('/', [HomeController::class, 'index']);
Route::get('open-source', [OpenSourceController::class, 'index']);

// Bad
Route::get('', [HomeController::class, 'index']);
Route::get('/open-source', [OpenSourceController::class, 'index']);
```

### Validation

**Use array syntax for validation rules:**
```php
// Good
public function rules()
{
    return [
        'email' => ['required', 'email'],
    ];
}

// Bad
public function rules()
{
    return [
        'email' => 'required|email',
    ];
}
```

**Custom validation rules use snake_case:**
```php
Validator::extend('organisation_type', function ($attribute, $value) {
    return OrganisationType::isValid($value);
});
```

### Authorization

**Policies and Gates use camelCase:**
```php
Gate::define('editPost', function ($user, $post) {
    return $user->id == $post->user_id;
});
```

## Naming Conventions

### Singular vs Plural

**Use plural form for:**
- Directories with multiple same-type classes (Actions, Models, Controllers, Events)
- Controllers and Resources (UsersController, CustomersResource)
- Actions operating on multiple entities (PatchOrdersAction - multiple, PatchOrderAction - single)

**Use singular form for:**
- Most other cases

### Domain Layer (app/Domain)

**Domains:**
- Use plural form
- Don't end with "Domain"
- Examples: `app/Domain/Customers`, `app/Domain/Orders`

**Models:**
- Singular form
- Don't end with "Model"
- Examples: `Customer`, `Order`

**Observers:**
- Name exactly like model + "Observer"
- Examples: `CustomerObserver`
- If `$afterCommit = true`: `CustomerAfterCommitObserver`

**Actions:**
- Start with verb + noun + "Action"
- Form: `{Verb}{Noun}Action`
- Use singular for single entity, plural for multiple
- Examples: `CreateCustomerAction`, `DeleteOrderAction`, `PatchProductsAction`
- Can add context: `SavePaymentFromExternalAction`

**Events:**
- Don't end with "Event"
- Show timing in name (before/after)
- Examples: `ApprovingProduct` (before), `ProductApproved` (after)

**Listeners:**
- Reflect action + "Listener"
- Examples: `SendInvitationMailListener`

**Mail:**
- End with "Mail"
- Examples: `AccountActivatedMail`, `NewEventMail`

### Http Layer (app/Http)

**Controllers:**
- Plural noun + "Controller"
- Examples: `UsersController`, `EventDaysController`

**Resources:**
- Plural noun + "Resource"
- Examples: `CustomersResource`

**Query Builder Requests:**
- Plural noun + "Query"
- Examples: `OrdersQuery`

**Form Requests:**
- Verb + noun + "Request"
- Examples: `CreateProductRequest`, `UpsertOrderItemsRequest`

**Tests:**
- Match controller name with "Controller" → "ComponentTest"
- Example: `UsersController` → `UsersComponentTest`

**Policies:**
- Match controller name with "Controller" → "ControllerPolicy"
- Example: `UsersController` → `UsersControllerPolicy`

### Console Layer

**Commands:**
- End with "Command"
- Examples: `PublishScheduledPostsCommand`

## When Applying This Skill

**Review existing code:**
1. Check naming conventions match layer-specific rules
2. Verify return types and property typing
3. Ensure proper constructor property promotion
4. Check if statement structure and early returns
5. Validate Laravel-specific patterns (routes, validation, config)

**Write new code:**
1. Start with correct naming based on layer
2. Apply PHP style rules (types, docs, whitespace)
3. Follow Laravel conventions for framework-specific code
4. Use proper structure for the layer (Domain vs Http)

**Refactor code:**
1. Identify style violations
2. Apply fixes systematically
3. Maintain code functionality while improving style
4. Run tests after changes

## Common Pitfalls to Avoid

- ❌ Using `env()` outside config files - use `config()`
- ❌ Naming directories with "Domain" suffix - just use the domain name
- ❌ Naming models with "Model" suffix - just use the entity name
- ❌ Using pipe-separated validation rules - use arrays
- ❌ Using string concatenation - use interpolation
- ❌ Deep nesting and else statements - use early returns
- ❌ Excessive PHPDoc - rely on type hints
- ❌ Multiple traits on one use line - one per line

Remember: **Existing codebase conventions always override these guidelines**. When working with established projects, maintain consistency with the existing codebase even if it differs from this style guide.
