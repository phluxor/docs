# Context

Phluxor has two contexts,  
`Phluxor\ActorSystem\ActorContext` and `Phluxor\ActorSystem\RootContext`.

These two contexts implement common features such as `Spawner`, `Stopper`, `Infomation`, and `Sender`.

In addition, `ActorContext` implements additional features.  

## Spawner

`Phluxor\ActorSystem\ActorContext` and `Phluxor\ActorSystem\RootContext` have a `spawn` method.  

`Spawner` is a feature that creates a new actor from the `Phluxor\ActorSystem\Props` object.  
The `Props` object is a configuration object for creating actors.  

```php
<?php

use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

$system = ActorSystem::create();

$spawned = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
```

see also [Props](/en/features/props.html) - what are props?

## Stopper

`Phluxor\ActorSystem\ActorContext` and `Phluxor\ActorSystem\RootContext` have a `stop` method.

A feature that allows you to stop an actor immediately or instruct it to stop after processing the current mailbox message.

`stop` and `poison` have several methods,  
but these methods require a `Phluxor\ActorSystem\Ref` object as an argument.

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        $context->stop($context->sender());
        $context->stop($context->self());
        $context->stop(new Ref(new ActorSystem\ProtoBuf\Pid([
            'id' => 'actor name',
        ])));
    }
}
```

## Information

The `info` is not a method, but a feature that provides information about the actor.  
`Phluxor\ActorSystem\ActorContext` and `Phluxor\ActorSystem\RootContext` have information features.

Context information can be obtained,  
such as the parent actor of the current actor,  
the `Phluxor\ActorSystem\Ref` of the current actor,  
the `Phluxor\ActorSystem\Ref` of the message sender actor,  
and the `Phluxor\ActorSystem` where the actor exists.

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message(); // get message
        $context->sender(); // get sender Ref
        $context->self(); // get self Ref
        $contex->parent(); // get parent Ref
        $context->actorSystem(); // get actor system
        $context->logger(); // get logger
        // etc...
    }
}
```

## Sender

`Phluxor\ActorSystem\ActorContext` and `Phluxor\ActorSystem\RootContext` have a `sender` method.

The `send` method sends a fire-and-forget style message,  
and the `request` method provides the ability to asynchronously request a response from the destination actor.  

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    // implement receive method
    public function receive(ContextInterface $context): void
    {
        // example - make a actor reference
        $pid = new Ref(new Pid([
            'id' => 'otherActor',
        ]));
        // Of course, 
        // you can also use the spawn method to create an actor reference.
        // fire-and-forget style message
        $context->send($pid, new YourMessage('hello'));
        // request style message
        $context->request($pid, new YourMessage('hello'));
        // future style message
        $context->requestFuture($pid, new YourMessage('hello'), 1);
    }
}
```

## Receiver

only `Phluxor\ActorSystem\ActorContext`.  

The `receive` method is a feature that provides the ability to receive a message wrapped in a `Phluxor\ActorSystem\Message\MessageEnvelope`.  

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    // implement receive method
    public function receive(ContextInterface $context): void
    {
        $context->receive($envelope);
    }
}
```

## Supervisor

only `Phluxor\ActorSystem\ActorContext`.

Supervision provides methods for controlling the lifecycle of child actors under supervision of the current actor  
and the ability to escalate failures up to the next actor in a supervision hierarchy.  
