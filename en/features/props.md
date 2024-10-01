# Props

The Props are a configuration object used to create an actor.  

`Phluxor\ActorSystem\Props` is a class that creates a Props object.  

## Basic Usage

### fromProducer

`fromProducer()` is a static method that creates a Props object.  

```php
Props::fromProducer(fn() => new YourActor())
```

The `fromProducer()` method creates a Props object with a producer.  
first parameter is a closure that returns an actor instance (`Closure(): ActorInterface`).  
or implement the `ProducerInterface` interface.  

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

interface ProducerInterface
{
    /**
     * @return ActorInterface
     */
    public function __invoke(): ActorInterface;
}

```

### fromFunction

`fromFunction()` is a static method that creates a Props object.  

```php
Props::fromFunction(
    new ActorSystem\Message\ReceiveFunction(
        fn(ActorSystem\Context\ContextInterface $context) => $context->message()
    )
);
```

The `fromFunction()` method creates a Props object with a function.  
first parameter is a `Phluxor\ActorSystem\Message\ReceiveFunction` object.  

`ReceiveFunction` is a class that wraps a function.  
The function receives a `Phluxor\ActorSystem\Context\ContextInterface` object.  

## Custom Props

Optionally, you can create a custom Props object.  

all options are second and later arguments.  

### withDispatcher

`withDispatcher()` is a method that sets the dispatcher for the Props object.

The dispatcher is used to dispatch messages to actors.

Customization is not recommended unless you have sufficient knowledge of the actor model or Phluxor.

```php
Props::fromProducer(
    fn() => new YourActor(),
    Props::withDispatcher(new ActorSystem\Dispatcher\SynchronizedDispatcher())
);
```

the default dispatcher uses the `Phluxor\ActorSystem\Dispatcher\CoroutineDispatcher` class.  
this dispatcher is optimized for the Swoole extension.  

if you want to use the default dispatcher, you don't need to set it.  

### withMailbox

`withMailbox()` is a method that sets the mailbox for the Props object.  

The mailbox is used to Queue messages for actors.  

```php
Props::fromProducer(
    fn() => new YourActor(),
    Props::withMailboxProducer(new ActorSystem\Mailbox\Unbounded())
);
```

the default mailbox uses the `Phluxor\ActorSystem\Mailbox\Unbounded` class.  
this mailbox is unbounded and can store an unlimited number of messages.  

### withSupervisor

`withSupervisor()` is a method that sets the supervisor for the Props object.  

the default strategy restarts child actors a maximum of 10 times within a 10 second window.

```php

Props::fromProducer(
    fn() => new VoidActor(),
    ActorSystem\Props::withSupervisor(
        new ActorSystem\Strategy\OneForOneStrategy(
            10,
            new \DateInterval('PT10S'),
            fn() => ActorSystem\Directive::Restart
        )
    )
);
```

the default supervisor uses the `Phluxor\ActorSystem\Strategy\OneForOneStrategy` class.  

if you want to use the default supervisor, you don't need to set it.  

### withReceiverMiddleware

receive middlewares are invoked before the actor receives the message.  

```php

Props::fromProducer(
    fn() => new VoidActor(),
    Props::withReceiverMiddleware(
        $this->mockReceiverMiddleware(
            function (ContextInterface $context, MessageEnvelope $messageEnvelope) {
                if ($messageEnvelope->getMessage() === 'hello') {
                    // logging and other processing
                }
            }
        )
    )
);

// example of mockReceiverMiddleware
private function mockReceiverMiddleware(Closure|ReceiverFunctionInterface $next): Props\ReceiverMiddlewareInterface
{
    return new readonly class($next) implements Props\ReceiverMiddlewareInterface {

        public function __construct(
            private Closure|ReceiverFunctionInterface $next
        ) {
        }

        public function __invoke(
            Closure|ReceiverFunctionInterface $next
        ): ReceiverFunctionInterface {
            return new readonly class($this->next) implements ReceiverFunctionInterface {

                public function __construct(
                    private Closure|ReceiverFunctionInterface $next
                ) {
                }
                
                public function __invoke(
                    ReceiverInterface|ContextInterface $context,
                    MessageEnvelope $messageEnvelope
                ): void {
                    $next = $this->next;
                    $next($context, $messageEnvelope);
                }
            };
        }
    };
}
```

For example, you can log messages or execute other processes.

It can also be used as a persistence middleware to save events in a database.

If you are implementing CQRS/ES, it is recommended to use persistence middleware.
