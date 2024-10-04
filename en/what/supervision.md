# Supervision

Supervision is the functionality where a parent actor supervises the state of its child actors,  
and monitoring is one of the features supported by actor model toolkits,  
allowing one actor to monitor the state of another actor.

This might be hard to imagine when thinking about typical web application frameworks or common processing flows,  
but when an actor terminates abnormally,  
the actor that is monitoring it can be notified of the abnormal termination.

Additionally, the parent actor can specify how to restart the child actor.

So far, we've briefly explained the hierarchy, but this hierarchy is closely related to supervision,  
and fully understanding it is the first step to grasping the actor model.

Now, let's explain the concepts of supervision and monitoring, and what monitoring means in Phluxor.

## What Supervision Means

A parent actor can delegate tasks to its child actors while also bearing the responsibility to handle the failures of those child actors (becoming a supervisor).

(Akka, Pekko, and others have similar supervision and monitoring functionality.)

The child actors referred to here are all the actors to which tasks are delegated and that are created (spawned) by the parent actor itself.

When a failure of a child actor is detected (in Phluxor, an Exception is detected),  
the parent actor suspends itself and all of its child actors, and sends a message notifying of the failure.

There are four options for handling this:

- Resume: Resume the actor while retaining its accumulated internal state.
- Restart: Restart the actor, clearing its accumulated internal state.
- Stop: Permanently stop the actor, preventing it from processing any further messages.
- Escalate: Escalate the failure to the parent actor within the hierarchy.

In the actor model, actors must always be considered as part of a hierarchy.

As you implement how to handle failures, consider how you will instruct child actors when failures occur.

It’s perfectly fine to simply restart the actor for now!

Unlike traditional PHP applications where exceptions are caught and handled,

in the actor model it’s important to receive failures and then decide on a strategy to handle them.

Let’s explain what to keep in mind.

The following behaviors should be understood when dealing with actors within a hierarchy:

The supervisor is configured to convert detected exceptions into one of the four options listed above.

You may want to apply different strategies to specific actors,  
and since you can set the supervision strategy (called a supervisor strategy) when creating an actor,  
you can create actors with different strategies in advance.

Please note that in Phluxor, only parent supervision is supported.

Actors can only be created by other actors (parents), and top-level actors are provided by the actor system.

Therefore, the parent actor serves as the supervisor of its child actors.

It is guaranteed that an actor will not be supervised by anything external,  
so you can rest assured that unexpected errors will not be caught from unintended places.

Below is an example of a parent actor detecting the termination of a child actor.

The actor that stops will be as shown in the following example.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    /**
     * @throws \Exception
     */
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                $context->stop($context->self());
                break;
        }
    }
}

```

The parent actor that receives the termination notice will be as shown in the following example.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;
use Phluxor\ActorSystem\ProtoBuf\Terminated;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(Props::fromProducer(fn() => new ChildActor()));
                $context->send($ref, $message);
                break;
            case $message instanceof Terminated:
                $context->logger()->info('terminated', [
                    'who' => $message->getWho()->getId(),
                    'why' => $message->getWhy(),
                ]);
                break;
        }
    }
}

```

When the child actor is terminated, the parent actor receives a Terminated message, and the following will be displayed on the console.

![watch](/images/supervision/parent_watch.png "watch")

If the actor stops due to an Exception,  
it will be detected as shown below, and recovery will take place.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                // throw exception!
                throw new \Exception('hi, I am an exception');
                break;
        }
    }
}

```

![recover](/images/supervision/actor_recover.png "recover")

## What Lifecycle Monitoring Means

As we've explained so far, actors may start to appear more like something organic, rather than a traditional application or object.

The lifecycle of an actor represents the entire flow from the moment the actor is created until it is terminated.

First, an actor is fully alive upon creation. Then, by stopping or other means, the actor transitions into a dead state. Normally, only the parent, acting as the supervisor, understands this state.

However, this transition from an "alive" state to a "dead" state can be utilized in monitoring.

While supervision reacts to failures, monitoring can be used to react to this "dead" state, i.e., the termination of an actor.

Monitoring the transition from an "alive" state to a "dead" state is called lifecycle monitoring.

Lifecycle monitoring can be implemented using the `Phluxor\ActorSystem\ProtoBuf\Terminated` message,  
which the monitoring actor receives.

To receive the `Phluxor\ActorSystem\ProtoBuf\Terminated` message,  
you need to start monitoring by calling `$context->watch($targetRef)`.

Lastly, to stop monitoring, you just need to call `$context->unwatch($targetRef)`.

The important thing to note is that the message will be delivered regardless of the order in which the monitoring request and the termination of the target occur.

In other words, you will still receive the message even if the target is in a stopped state at the time of registration.

If the supervisor actor is unable to simply restart a child actor—such as when an error occurs during the initialization of the actor—it may be necessary to terminate the child actor.

This is where monitoring can be especially useful.

The supervisor can monitor its child actor and, once it has terminated, can either recreate it or schedule a retry later.

As an example, consider a web server actor.

The web server actor processes requests using multiple child actors, but if the database connection fails during initialization, simply restarting the child actor won’t solve the problem.

In this case, the supervisor can monitor the termination of the child actor and schedule a retry for the database connection.

Another common use case is when an actor must fail due to the absence of an external resource.

This external resource could be a child actor of the main actor.

For example, if a supervisor actor has a database connection actor, and a third party terminates that child actor using $context->stop($ref) or $context->poison($ref), the supervisor might be affected by this.

In such situations, monitoring would be particularly helpful.

## Supervisor Strategy

We’ve explained monitoring so far, but you can also implement how actors are restarted in case of failures,  
such as those caused by external resources.

For example, if an actor crashes due to the high load on a resource like a database,  
there are strategies that allow for gradually spacing out the time between restarts and reconnections.

### ExponentialBackoffStrategy

`ExponentialBackoffStrategy` is a strategy for implementing an exponential backoff approach.

This strategy increases the delay exponentially each time the actor is restarted.

```php
new Phluxor\ActorSystem\Strategy\ExponentialBackoffStrategy(
    new DateInterval("PT02S"),
    new DateInterval("PT01S")
);
```

The first argument is `backoffWindow` (DateInterval), which specifies the time frame in which restarts should occur.

The second argument is `initialBackoff` (DateInterval), which defines the time delay before the first restart attempt.

An actual usage example is shown below.

Here is an example that throws an exception when creating a child actor, so that the restart can be easily observed.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Started:
                throw new \Exception('hi, I am an exception');
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                break;
        }
    }
}
```

Next, specify the use of the **ExponentialBackoffStrategy** strategy for the parent actor.

```php
<?php

declare(strict_types=1);

use Phluxor\ActorSystem;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();
        $ref = $system->root()->spawn(
            ActorSystem\Props::fromProducer(
                fn() => new Example\ParentActor(),
                ActorSystem\Props::withSupervisor(
                    new ActorSystem\Strategy\ExponentialBackoffStrategy(
                        new DateInterval("PT02S"),
                        new DateInterval("PT01S")
                    )
                )
            )
        );
        $system->root()->send(
            $ref,
            new Example\Message\Hello('World')
        );
    });
});
```

If the restarts exceed the time frame specified by backoffWindow,  
the restart time for the actor will be reset, and restarts will begin again from the start.

Once the actor has stabilized after a restart, it will resume receiving messages.

However, note that it will not reprocess the message that was being handled when the Exception was thrown,  
so you will need to resend the message from the parent actor.

## OneForOneStrategy and AllForOneStrategy

`OneForOneStrategy` and `AllForOneStrategy` are strategies used to specify how the supervisor responds to child actor failures.

Both strategies have a limit on the number of failures a child actor can experience before it is completely stopped.

`OneForOneStrategy` applies an individual strategy to the failed child actor.

`AllForOneStrategy` applies the same strategy to all child actors within the same hierarchy.

The default is `OneForOneStrategy`, but you should choose between them depending on the use case or load balancing strategy.

When using **OneForOneStrategy**,  
you can utilize `Phluxor\ActorSystem\Strategy\OneForOneStrategy`. Unlike ExponentialBackoffStrategy,  
this strategy allows you to select directives other than just restarting.

The available directives have the following differences:

### Phluxor\ActorSystem\Resume

Instructs the actor that encountered an error to resume and continue processing messages.

The error is ignored, and the actor continues processing messages in its current state.

### Phluxor\ActorSystem\Restart

Instructs the actor that encountered an error to be discarded and replaced with a new instance (restart).

The actor’s state is reinitialized, and after the restart, it continues to operate as a new instance.

### Phluxor\ActorSystem\Stop

Instructs the actor that encountered an error to stop.

The stopped actor ceases processing messages and is removed from the system.

### Phluxor\ActorSystem\Escalate

Instructs the error to be escalated to the actor’s parent.

With this directive, the error handling is passed to the parent actor.

To set up a strategy for handling actor failures using these directives,

you need to either use a **Closure** or implement the `Phluxor\ActorSystem\Supervision\DeciderFunctionInterface` as shown below.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Supervision;

use Phluxor\ActorSystem\Directive;

interface DeciderFunctionInterface
{
    /**
     * @param mixed $reason
     * @return Directive
     */
    public function __invoke(mixed $reason): Directive;
}
```

`$reason` will contain the message sent by the failed actor, which in most cases will be an Exception.

```php
Phluxor\ActorSystem\Props::withSupervisor(
    new Phluxor\ActorSystem\Strategy\OneForOneStrategy(
        15,
        new DateInterval('PT1S'),
        function (mixed $reason): ActorSystem\Directive {
            return Phluxor\ActorSystem\Directive::Restart;
        }
    )
)                
```

In this example, the settings allow for 15 restarts per second.

Since an Exception is passed to `$reason`,

you can adjust the appropriate directive based on the exception or message content.

If you are using **AllForOneStrategy**, please use `Phluxor\ActorSystem\Strategy\AllForOneStrategy`.
