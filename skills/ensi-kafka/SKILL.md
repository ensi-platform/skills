---
name: ensi-kafka
description: Work with Kafka in Ensi Laravel services following project standards. Use this skill whenever the user mentions creating Kafka producers, consumers, observers, or any Kafka-related work in Ensi context. This includes requests like "create a Kafka producer for Product entity", "set up Kafka consumer", "add observer to send events", "create Kafka payload", "configure Kafka topics", or any work related to Kafka integration, message producers/consumers, observers, or event-driven architecture in Ensi Laravel projects.
---

# Ensi Kafka Integration Guide

This skill helps you work with Kafka in Ensi Laravel services. It provides guidance on creating producers, understanding architecture, setting up consumers, and testing Kafka integration.

## Architecture Overview

### Directory Structure

```
app/Domain/Kafka/
├── Actions/Send/
│   ├── SendMessageAction.php          (Base class)
│   └── Send{Entity}EventAction.php   (Entity-specific actions)
├── Messages/Send/
│   ├── KafkaMessage.php              (Base class)
│   ├── Payload.php                   (Base class)
│   └── ModelEvent/
│       ├── ModelEventMessage.php     (Model event message)
│       └── {Entity}Payload.php       (Entity payloads)
```

### Key Concepts

- **SendMessageAction**: Base class for sending messages to Kafka topics
- **KafkaMessage**: Abstract base class for all Kafka messages
- **Payload**: Base class for message payloads (JSON serializable)
- **ModelEventMessage**: Standard message format for model events (create/update/delete)
- **Observers**: Laravel observers that trigger Kafka events on model changes

### Ensi Packages Used

- `ensi/laravel-phprdkafka` - Core Kafka integration
- `ensi/laravel-phprdkafka-producer` - Producer functionality
- `ensi/laravel-phprdkafka-consumer` - Consumer functionality
- `ensi/laravel-initial-event-propagation` - Event tracking across services
- `ensi/laravel-metrics` - Kafka metrics and monitoring

### Configuration Files

- **config/kafka.php** - Connection settings, topics, consumers, producers
- **config/kafka-producer.php** - Producer middleware configuration
- **config/kafka-consumer.php** - Consumer configuration and middleware

### Topic Naming Convention

Topics follow this pattern: `{contour}.{app_name}.fact.{entity_name}.1`

Examples:
- `local.cms.fact.nameplates.1`
- `prod.cms.fact.banner.1`
- `dev.cms.fact.products.1`

### Event Types

Standard model events:
- `create` - Model created
- `update` - Model updated (includes dirty fields tracking)
- `delete` - Model deleted

---

## Creating Kafka Producers

### Step 1: Create Payload Class

Create a payload class that extends `Payload` in `app/Domain/Kafka/Messages/Send/ModelEvent/`:

```php
<?php

namespace App\Domain\Kafka\Messages\Send\ModelEvent;

use App\Domain\Kafka\Messages\Send\Payload;
use App\Domain\{YourDomain}\Models\{YourModel};

class {YourModel}Payload extends Payload
{
    public function __construct(protected {YourModel} $model) {}

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->model->id,
            'name' => $this->model->name,
            'code' => $this->model->code,
            'created_at' => $this->model->created_at?->toJSON(),
            'updated_at' => $this->model->updated_at?->toJSON(),
        ];
    }
}
```

**Best Practices:**
- Include all fields that consumers might need
- Use `->toJSON()` for timestamps
- Keep payloads minimal but complete
- Use nullable operators (`?->`) for potentially null timestamps

### Step 2: Create Send Action Class

Create an action class in `app/Domain/Kafka/Actions/Send/`:

```php
<?php

namespace App\Domain\Kafka\Actions\Send;

use App\Domain\Kafka\Messages\Send\ModelEvent\ModelEventMessage;
use App\Domain\Kafka\Messages\Send\ModelEvent\{YourModel}Payload;
use App\Domain\{YourDomain}\Models\{YourModel};

class Send{YourModel}EventAction extends SendMessageAction
{
    public function execute({YourModel} $model, string $event): void
    {
        $modelEvent = new ModelEventMessage(
            $model,
            new {YourModel}Payload($model),
            $event,
            '{topic_key}'  // Must match topic key in config/kafka.php
        );
        $this->send($modelEvent);
    }
}
```

**Important Notes:**
- Topic key must match a key in `config/kafka.php` under `connections.default.topics`
- Follow naming convention: `Send{Entity}EventAction`
- The action is automatically available for dependency injection

### Step 3: Add Topic Configuration

Add your topic to `config/kafka.php`:

```php
'topics' => [
    'existing_topic' => $contour . '.cms.fact.existing_entity.1',
    '{topic_key}' => $contour . '.cms.fact.{entity_name}.1',
],
```

**Topic Key vs Topic Name:**
- **Topic key**: Internal identifier used in code (e.g., `nameplates`, `banner`)
- **Topic name**: Full Kafka topic name with contour and version (e.g., `local.cms.fact.nameplates.1`)

### Step 4: Create Observer

Create an observer in `app/Domain/{YourDomain}/Observers/`:

```php
<?php

namespace App\Domain\{YourDomain}\Observers;

use App\Domain\Kafka\Actions\Send\Send{YourModel}EventAction;
use App\Domain\Kafka\Messages\Send\ModelEvent\ModelEventMessage;
use App\Domain\{YourDomain}\Models\{YourModel};

class {YourModel}AfterCommitObserver
{
    public bool $afterCommit = true;

    public function __construct(protected Send{YourModel}EventAction $eventAction) {}

    public function created({YourModel} $model): void
    {
        $this->eventAction->execute($model, ModelEventMessage::CREATE);
    }

    public function updated({YourModel} $model): void
    {
        $this->eventAction->execute($model, ModelEventMessage::UPDATE);
    }

    public function deleted({YourModel} $model): void
    {
        $this->eventAction->execute($model, ModelEventMessage::DELETE);
    }
}
```

**Key Points:**
- Set `$afterCommit = true` to ensure events are sent only after database transaction commits
- Only include methods for events you want to track
- The observer is automatically registered by Laravel

### Step 5: Register Observer in EventServiceProvider

Add to `app/Providers/EventServiceProvider.php`:

```php
use App\Domain\{YourDomain}\Observers\{YourModel}AfterCommitObserver;

protected $observers = [
    {YourModel}::class => {YourModel}AfterCommitObserver::class,
];
```

---

## Creating Kafka Consumers

### Step 1: Create Processor Class

Create a processor class to handle incoming messages:

```php
<?php

namespace App\Domain\{YourDomain}\Kafka;

use Ensi\LaravelPhpRdKafkaConsumer\ProcessorInterface;
use RdKafka\Message;

class {YourModel}Processor implements ProcessorInterface
{
    public function process(Message $message): void
    {
        $data = json_decode($message->payload, true);

        if (!isset($data['event'], $data['attributes'])) {
            throw new \InvalidArgumentException('Invalid message format');
        }

        $event = $data['event'];
        $attributes = $data['attributes'];
        $dirty = $data['dirty'] ?? null;

        switch ($event) {
            case 'create':
                $this->handleCreate($attributes);
                break;
            case 'update':
                $this->handleUpdate($attributes, $dirty);
                break;
            case 'delete':
                $this->handleDelete($attributes);
                break;
            default:
                throw new \InvalidArgumentException("Unknown event type: {$event}");
        }
    }

    protected function handleCreate(array $attributes): void
    {
        // Handle creation event
    }

    protected function handleUpdate(array $attributes, ?array $dirty): void
    {
        // Handle update event - $dirty contains changed field names
    }

    protected function handleDelete(array $attributes): void
    {
        // Handle deletion event
    }
}
```

### Step 2: Configure Consumer

Add processor to `config/kafka-consumer.php`:

```php
'processors' => [
    [
        'topic' => '{contour}.{service}.fact.{entity}.1',
        'consumer' => 'default',
        'processor' => \App\Domain\{YourDomain}\Kafka\{YourModel}Processor::class,
    ],
],
```

### Consumer Best Practices

1. **Error Handling**: Always validate message structure
2. **Idempotency**: Handle duplicate messages gracefully
3. **Logging**: Log all processing events for debugging
4. **Metrics**: Track processing success/failure rates
5. **Dead Letter Queue**: Configure for failed messages

---

## Testing Kafka Integration

### Unit Testing Messages

Test payload classes and message structure:

```php
<?php

use App\Domain\Kafka\Actions\Send\Send{YourModel}EventAction;
use App\Domain\Kafka\Messages\Send\ModelEvent\{YourModel}Payload;
use App\Domain\{YourDomain}\Models\{YourModel};

use function PHPUnit\Framework\assertEquals;

use Tests\IntegrationTestCase;

uses(IntegrationTestCase::class);
uses()->group('unit');

test("generate {YourModel}Payload success", function () {
    $this->mock(Send{YourModel}EventAction::class)->shouldReceive('execute');

    /** @var {YourModel} $model */
    $model = {YourModel}::factory()->create();
    $payload = new {YourModel}Payload($model);

    assertEquals($payload->jsonSerialize(), $model->attributesToArray());
});
```

### Integration Testing Producers

Test full producer flow:

```php
test("{YourModel} observer sends Kafka message on create", function () {
    $action = mock(Send{YourModel}EventAction::class);
    $action->shouldReceive('execute')
        ->once()
        ->with(\Mockery::type({YourModel}::class), ModelEventMessage::CREATE);

    app()->instance(Send{YourModel}EventAction::class, $action);

    $model = {YourModel}::factory()->create();

    expect($model)->exists->toBeTrue();
});
```

### Testing Consumers

Test processor logic:

```php
test("{YourModel}Processor handles create event", function () {
    $processor = new {YourModel}Processor();

    $message = new \RdKafka\Message();
    $message->payload = json_encode([
        'event' => 'create',
        'attributes' => [
            'id' => 1,
            'name' => 'Test',
        ],
        'dirty' => null,
    ]);

    $processor->process($message);

    expect({YourModel}::where('id', 1)->exists())->toBeTrue();
});
```

### Test File Location

Place tests in:
- app/Domain/Kafka/Actions/Listen/Tests

---

## Best Practices

### Payload Design

1. **Include ID**: Always include entity ID
2. **Minimal Fields**: Only send what consumers need
4. **Relationships**: Flatten or nest based on consumer needs
5. **Data Types**: Use proper JSON types (booleans, integers, strings)

### Observer Design

1. **After Commit**: Use `$afterCommit = true` when needed
2. **Selective Events**: Only observe events you need
3. **Dependency Injection**: Inject actions via constructor

---

## Troubleshooting

### Common Issues

**Messages not being sent:**
1. Review logs for errors
2. Verify topic key matches configuration
3. Check Kafka broker connectivity
4. Check `$afterCommit` is true in observer

**Consumer not processing messages:**
1. Verify consumer group ID is unique
2. Check topic configuration in kafka-consumer.php
3. Verify message format matches expectations
4. Check consumer offset position

---

## Quick Reference

### Producer Checklist

- [ ] Create Payload class extending `Payload`
- [ ] Create Action class extending `SendMessageAction`
- [ ] Add topic to `config/kafka.php`
- [ ] Create observer
- [ ] Register observer in EventServiceProvider
- [ ] Write unit tests
- [ ] Test with actual model creation/updates/deletions

### Consumer Checklist

- [ ] Create Processor class implementing `ProcessorInterface`
- [ ] Add processor to `config/kafka-consumer.php`
- [ ] Configure consumer group ID
- [ ] Add middleware for metrics and event propagation
- [ ] Test processor with sample messages
- [ ] Set up monitoring and alerting

### Testing Checklist

- [ ] Unit tests for all payloads
- [ ] Integration tests for observers
- [ ] Consumer processor tests
- [ ] Failure scenario testing
