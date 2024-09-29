# Actors

Actors are the basic unit of the actor model.  

An actor is an object that encapsulates state and behavior,  
Mailbox for receiving messages,  
Children actors,  
and a supervisor strategy for handling errors.  

All actors have a unique address,  
Behind an Actor Reference,  
and communicate with other actors by sending messages.  

Every actor has a unique address and communicates  
with other actors by sending messages using only the actor reference[Ref].  

The `Phluxor\ActorSystem` class is the entry point for creating actors.  
*required swoole extension*

boot an actor system.  

```php
$system = \Phluxor\ActorSystem::create();
```

The actor system is the central component that manages the lifecycle of actors.  

## Implementing an Actor

actors are objects that implement the `Phluxor\ActorSystem\Message\ActorInterface` interface.  
all actors have a `receive` method that receives a context.  

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\ContextInterface;

interface ActorInterface
{
    /**
     * Receives a context.
     *
     * @param ContextInterface $context The context to receive.
     * @return void
     */
    public function receive(ContextInterface $context): void;
}

```

The `ContextInterface` interface provides access to the actor's mailbox.  

The context also provides access to the actor system, as well as information such as the actor's own address,  
parent actor details, child actor details, and supervisor strategy.

Please note that actors do not have a return value and always return void.

Since all actors operate asynchronously and concurrently, they cannot have return values.

Actors communicate with other actors through asynchronous message passing,

so you cannot directly call or manipulate an actor.
