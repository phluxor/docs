# Hello World

This guide explains how to create a simple "Hello World" using Phluxor,  
and how to understand the basics of actor systems, actors, and messages.  

## Create Actor System

An actor system is the central component that manages the lifecycle of actors.  
It creates and monitors actors.  

```php
\Swoole\Coroutine\run(function () {
    \Swoole\Coroutine\go(function () {
        $system = \Phluxor\ActorSystem::create();

    });
});
```

To create an actor system, use the ActorSystem::create() method.  
This will create the actor system, allowing you to create actors and send and receive messages.

A coroutine is used to create an actor system.  
This is because the actor system is a long-running process that manages actors.  

## Create Message

Actors communicate with other actors by sending messages. Here, we will create a message called Hello.  

```php
namespace Phluxor\Examples;

class Hello
{
    public function __construct(
        public readonly string $who
    ) {
    }
}
```

The Hello class is a message that has a who property.  
This message is used to communicate with other actors.

## Create Actor

Actors are entities that receive and process messages.  
(In the actor model, everything is an actor.)

Here, we will create an actor called HelloWorldActor.  
When it receives a Hello message,  
it will output the value of the who property.

```php
namespace Phluxor\Examples;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;

class HelloWorldActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        match (true) {
            $msg instanceof Hello => $context->logger()->info(sprintf('Hello %s', $msg->who)),
            default => ''
        };
    }
}

```

To behave as an actor, you need to implement the ActorInterface.  
The receive() method is called when a message is received,  
allowing you to access information such as messages  
and loggers through the `Phluxor\ActorSystem\Context\ContextInterface`.  

## Register Actor

The generated actor is registered with the actor system.  
Actor creation is done using the spawn method.  
Register the actor by writing `$system->root()->spawn()`.

```php
\Swoole\Coroutine\run(function () {
    \Swoole\Coroutine\go(function () {
        $system = \Phluxor\ActorSystem::create();
        $ref = $system->root()->spawn(
            \Phluxor\ActorSystem\Props::fromProducer(fn() => new \PhluxorExample\HelloWorldActor())
        );
    });
});
```

The `spawn()` method generates an actor and returns a reference to the actor.  
This reference serves as the address or identifier for recognizing the individual actor within the actor system.  
The address information can be represented as a string,  
so it can be converted to a string and outputted.

## Send a message

Create a `Hello` message and send it to the actor.  
Use the `$system->root()->send()` method to send the message.

```php
\Swoole\Coroutine\run(function () {
    \Swoole\Coroutine\go(function () {
        $system = \Phluxor\ActorSystem::create();
        $ref = $system->root()->spawn(
            \Phluxor\ActorSystem\Props::fromProducer(fn() => new \PhluxorExample\HelloWorldActor())
        );
        $system->root()->send($ref, new \Phluxor\Example\Hello('World'));
    });
});
```

we have created an actor called `HelloWorldActor` that outputs the message `Hello World`.

![Hello World](/images/hello_world.png "Hello World")
