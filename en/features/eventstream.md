# EventStream

Phluxor provides a built-in `EventStream` that allows actors to publish events and other actors to subscribe to those events.

The **EventStream** can be accessed by calling `actorSystem()->getEventStream()` from each actor.
(From the root of the ActorSystem, you can call `getEventStream()` directly.)

The **EventStream** allows any actor within the actor system to subscribe to any message.

Of course, just like a typical PubSub system, multiple actors can subscribe at the same time.

To subscribe, use the `subscribe` method of the EventStream class, and receive messages through a callback.

![flow](/images/eventstream/eventstream.png "eventstream")

```php
<?php

declare(strict_types=1);

namespace Example;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;

class SubscribeTwoActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $subscription = $context->actorSystem()->getEventStream()->subscribe(
            function ($message) use ($context) {
                $context->logger()->info('Received message', ['message' => $message]);
            }
        );
    }
}

```

If a subscription is no longer needed, use the unsubscribe method of the EventStream class to remove the subscription.  

```php
$subscription = $context->actorSystem()->getEventStream()->subscribe(
    function ($message) use ($context) {
        $context->logger()->info('Received message', ['message' => $message]);
    }
);
$context->actorSystem()->getEventStream()->unsubscribe($subscription);
```

The message can be any type of message. For example, it can be a simple message like the ones shown below,  
or a message that can be accessed from the actor context.

```php
<?php

declare(strict_types=1);

namespace Example\Message;

readonly class Ordered
{
    public function __construct(
        public string $dish
    ) {}
}
```

Publishing a message on the EventStream is simple. Just call the publish method.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Ordered;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;

class PublishActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        if ($message === 'order') {
            $context->actorSystem()->getEventStream()->publish(
                new Ordered('Pizza')
            );
        }
    }
}
```

If you want to quickly check how it works, you can try it with the following code.

```php
<?php

declare(strict_types=1);

use Example\PublishActor;
use Example\SubscribeActor;
use Example\SubscribeTwoActor;
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();

        $ref = $system->root()->spawn(Props::fromProducer(fn() => new PublishActor()));
        $system->root()->spawn(Props::fromProducer(fn() => new SubscribeActor()));
        $system->root()->spawn(Props::fromProducer(fn() => new SubscribeTwoActor()));
        $system->root()->send($ref, 'order');
    });
});
```
