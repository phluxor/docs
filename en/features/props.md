# Props

The Props are a configuration object used to create an actor.  

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
    fn() => new Fizz(),
    Props::withDispatcher(new ActorSystem\Dispatcher\SynchronizedDispatcher())
);
```

the default dispatcher uses the `Phluxor\ActorSystem\Dispatcher\CoroutineDispatcher` class.  
this dispatcher is optimized for the Swoole extension.  

if you want to use the default dispatcher, you don't need to set it.  

`ActorSystem\Dispatcher\SynchronizedDispatcher` is a dispatcher that processes messages synchronously.  
this dispatcher is not optimized for the Swoole extension.  
small resources are used, but performance is poor.  

