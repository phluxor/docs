# Ref / Actor Referece

In a typical actor model toolkit, you cannot directly reference an actor when you create it.  

Instead, you get a reference to the actor's mailbox and communicate with the actor by sending messages to that mailbox.  
This reference is called a `Phluxor\ActorSystem\Ref`,  
a.k.a `Actor Reference`.

This `Ref` is serializable and can be sent over the network at low cost.  
Whether local or remote, you can send messages to actors through Refs.  

`Phluxor\ActorSystem\Ref` to reference an actor.  


## Overview

in Phluxor, actors communicate with each other by sending messages.  
so to send a message, an actor needs a reference to another actor.

![send message to actor](/images/ref/send_ref.png "send message to actor")

When sending a message to an actor, it is important to identify the actor.  
In Phluxor, a unique `Ref` is used to identify actors.  

When a new actor is created, a `Ref` to that actor is returned.  
Phluxor automatically assigns a name to the actor that is not assigned to any actor.  

When an actor stops, its `Ref` becomes invalid.  

![send message to actor with ref](/images/ref/send_to_actor_with_ref.png "send message to actor with ref")

## Create Actor

When an actor is spawned, a `Ref` is created and returned.  

### root

```php
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

$system = ActorSystem::create();

// spawn a root actor
// return a Ref
$ref = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
```

### child

```php
namespace App\ActorSystem;

use App\Command\CreateUser;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        if ($msg instanceof CreateUser) {
            // spawn a child actor
            // return a Ref
            $ref = $context->spawn(
                Props::fromProducer(
                    fn() => new YourChildActor()
                )
            );
        }
    }
}

```

## Create Manually

`Phluxor\ActorSystem\Ref` can be created manually.  

It is meant to act as an actor identifier,  
so there is no harm in manually creating and using it as long as the name is appropriate.  

Ref is created with a `Phluxor\ActorSystem\ProtoBuf\Pid` object.  
make sure to set the `address` to `ActorSystem::LOCAL_ADDRESS`.  

because the address is used to determine whether the actor is local or remote.  
if the address is not `ActorSystem::LOCAL_ADDRESS`, the actor is considered remote.  

```php
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\ProtoBuf\Pid;
use Phluxor\ActorSystem\Ref;

new Ref(new Pid([
    'id' => 'special-id',
    'address' => ActorSystem::LOCAL_ADDRESS,
]));
```

This `Ref` Object has a `__toString()` method,  
so you can convert it to a string and output the address.  

## Send Message

Send is a non-blocking, fire-and-forget method for sending a message to an actor. 

```php
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        switch (true) {
            case $msg instanceof StartsClass:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new TeacherActor(
                            $this->students, $context->self()
                        )
                    )
                );
                // send a message to the actor
                $context->send($ref, new PrepareTest(['subject' => $msg->getSubject()]));
                break;
    }
```

## Request Message

`request` is very similar to `send`,  
but it includes the sender's `Ref` so that the receiving actor can reply to the sender.  

Only use it when request/reply communication is required between two actors.  

```php
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        switch (true) {
            case $msg instanceof StartsClass:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new TeacherActor(
                            $this->students, $context->self()
                        )
                    )
                );
                // request a message to the actor
                $context->request($ref, new PrepareTest(['subject' => $msg->getSubject()]));
                break;
    }
```
