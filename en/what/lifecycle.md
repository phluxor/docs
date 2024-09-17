# Actor Lifecycle

The actor lifecycle in Phluxor represents the sequence of events from the creation to the termination of an actor. 

In a basic scenario, an actor is first instantiated,  
a Started message is sent,  
and the actor remains alive until the application shuts down.

## Stopping an Actor

You can stop an actor using various methods:

Sending a Stop message from the context.  

Calling the stop method on a `Phluxor\ActorSystem\ActorContext` instance.  

## Failure and supervision

If an actor fails to process a message,  
the actor's mailbox is temporarily paused to stop processing.

This failure is escalated to other actors or the parent actor supervising the failed actor (see supervision).  
Depending on the instructions decided by the supervisor,  
the actor may automatically resume, restart, or stop.

### Resume

The mailbox resumes processing,  
and the actor continues execution from the message that previously failed.

### Stop

When an actor is stopped,  
the object that was functioning as the actor is discarded.  
Before the stop, the actor receives a stop message, and after being stopped,  
it also receives the stop message.

### Restart

A restart message is sent to the actor,  
notifying it that a restart is being attempted.  
After this is processed, the actor stops, and the object that was functioning as the actor is discarded and recreated.  
The mailbox is then resumed,  
and the newly created actor receives a start message.  
Past messages cannot be retrieved after a restart.

## Handling events

`Phluxor\ActorSystem\Message\Started` is the first message an actor receives after being spawned or restarted.  
If you need to set the initial state of the actor,  
such as loading data from a database, you should handle this message.

`Phluxor\ActorSystem\Message\Restarting` is sent when an actor is about to restart,  
and `Phluxor\ActorSystem\Message\Stopping` is sent when an actor is about to stop.  
In both cases, the actor's object will be discarded,  
so if you need to execute any logic to ensure a proper shutdown (e.g., saving the state to a database, sending a message to a messaging system, etc.),  
you should handle these messages.

`Phluxor\ActorSystem\Message\Stopped` is sent when an actor has stopped,  
and the actor and its associated objects have been detached from the system.  
At this stage, the actor can no longer send or receive messages,  
and after the message is processed, the object is completely discarded.

## Flow

The flow of the actor lifecycle is as follows:  

![flow](/images/lifecycle/lifecycle_flow.png "Flows")

## Example

Let's quickly understand the lifecycle through a simple application.

First, we'll create a message that requests playing a movie.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample\Message;

readonly class PlayMovie
{
    public function __construct(
        public string $movie,
        public int $userId
    ) {
    }
}
```

This message is used to request the playback of a movie.

```php
$system->root()->send($ref, new PlayMovie('Transformers', 1));
$system->root()->send($ref, new PlayMovie('Transformers last knight', 2));
$system->root()->send($ref, new PlayMovie('Transformers age of extinction', 3));
$system->root()->send($ref, new PlayMovie('Transformers dark of the moon', 4));
$system->root()->send($ref, new PlayMovie('Transformers revenge of the fallen', 5));
```

Next, we'll create an actor that plays movies.

This actor will play a movie when it receives a PlayMovie message.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;
use PhluxorExample\Message\PlayMovie;

class PlaybackActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PlayMovie:
                $context->logger()->info("Playing movie $message->movie for user $message->userId");
                break;
        }
    }
}
```

To run this actor, you need to register it with the actor system and start it.

Let's create a `main.php` file like the following and try running it.

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

function main(): void
{
    run(function () {
        \Swoole\Coroutine\go(function () {
            $system = ActorSystem::create();
            $ref = $system->root()->spawn(
                ActorSystem\Props::fromProducer(
                    fn() => new PlaybackActor()
                )
            );
            $system->root()->send($ref, new PlayMovie('Transformers', 1));
            $system->root()->send($ref, new PlayMovie('Transformers last knight', 2));
            $system->root()->send($ref, new PlayMovie('Transformers age of extinction', 3));
            $system->root()->send($ref, new PlayMovie('Transformers dark of the moon', 4));
            $system->root()->send($ref, new PlayMovie('Transformers revenge of the fallen', 5));
        });
    });
}

main();
```

Please confirm that the movie titles and user IDs are output to the log.

From here, to understand the actor's lifecycle, let's modify it to respond to some internal messages.

## Started

`Phluxor\ActorSystem\Message\Started` is the first message sent after an actor is created.

We'll add processing to handle this message.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;
use Phluxor\ActorSystem\Message\Started;
use Phluxor\ActorSystem\Props;
use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;

class PlaybackActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Started:
                $context->logger()->info("PlaybackActor started");
                break;
            case $message instanceof PlayMovie:
                $context->logger()->info("Playing movie $message->movie for user $message->userId");
                break;
        }
    }
}

```

When you run this, you can confirm that `PlaybackActor started` is output to the log when the actor is created.

## Restarting

When a failure occurs in an actor,  
the actor system restarts the actor to fix the issue and sends a `Phluxor\ActorSystem\Message\Restarting` message to notify the actor about the upcoming restart.

Unlike the `Started` message, handling the `Restarting` message is slightly different.

Here, we'll add a `PhluxorExample\Message\Recover` message so that when a child actor receives this message,  
it will crash, detect the failure, and then restart.

First, let's create the `PhluxorExample\Message\Recover` message.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample\Message;

class Recover
{
}
```

Next, we'll create a `ChildActor` that crashes when it receives a `Recover` message.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;
use PhluxorExample\Message\Recover;

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
                $context->logger()->info("ChildActor restarting");
                break;
            case $message instanceof Recover:
                throw new \Exception('child actor exception');
        }
    }
}
```

Next, we'll have the `PlaybackActor` create a `ChildActor` and forward the `Recover` message to the child actor.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Started;
use Phluxor\ActorSystem\Props;
use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;

class PlaybackActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Started:
                $context->logger()->info("PlaybackActor started");
                break;
            case $message instanceof Recover:
                $this->recoverMessageHandler($context);
                break;
            case $message instanceof PlayMovie:
                $context->logger()->info("Playing movie $message->movie for user $message->userId");
                break;
        }
    }

    private function recoverMessageHandler(ContextInterface $context): void
    {
        if (count($context->children()) === 0) {
            $child = $context->spawn(Props::fromProducer(fn() => new ChildActor()));
        } else {
            $child = $context->children()[0];
        }
        $context->forward($child);
        \Swoole\Coroutine::sleep(0.1);
    }
}

```

In the `recoverMessageHandler` method,  
the parent actor creates a child actor and forwards the `Recover` message to the child actor.

After the child actor receives this message and crashes, it restarts,  
receives the `Phluxor\ActorSystem\Message\Restarting` message,  
and logs that it has restarted.

To execute this, modify the previous `main.php` as follows.

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

function main(): void
{
    run(function () {
        \Swoole\Coroutine\go(function () {
            $system = ActorSystem::create();
            $ref = $system->root()->spawn(
                ActorSystem\Props::fromProducer(
                    fn() => new PlaybackActor()
                )
            );
            $system->root()->send($ref, new PlayMovie('Transformers', 1));
            $system->root()->send($ref, new PlayMovie('Transformers last knight', 2));
            $system->root()->send($ref, new PlayMovie('Transformers age of extinction', 3));
            $system->root()->send($ref, new PlayMovie('Transformers dark of the moon', 4));
            $system->root()->send($ref, new PlayMovie('Transformers revenge of the fallen', 5));
            $system->getLogger()->info('restarting actor');
            $system->root()->send($ref, new Recover());
            $system->root()->send($ref, new PlayMovie('Transformers One', 6));
        });
    });
}

main();
```

Please confirm that the child actor has crashed and restarted, and that this has been logged.

Were you able to confirm it?

## Stopping

Just before an actor stops, the actor system sends a `Phluxor\ActorSystem\Message\Stopping` message.

Stopping is used to release resources, perform cleanup, disconnect from external resources, and so on.

To implement this, we add processing to handle the `Phluxor\ActorSystem\Message\Stopping` message.

```php
<?php

declare(strict_types=1);

namespace PhluxorExample;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Started;
use Phluxor\ActorSystem\Message\Stopping;
use Phluxor\ActorSystem\Props;
use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;

class PlaybackActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Started:
                $context->logger()->info("PlaybackActor started");
                break;
            case $message instanceof Recover:
                $this->recoverMessageHandler($context);
                break;
            case $message instanceof Stopping:
                $context->logger()->info("PlaybackActor stopping");
                break;
            case $message instanceof PlayMovie:
                $context->logger()->info("Playing movie $message->movie for user $message->userId");
                break;
        }
    }

    private function recoverMessageHandler(ContextInterface $context): void
    {
        if (count($context->children()) === 0) {
            $child = $context->spawn(Props::fromProducer(fn() => new ChildActor()));
        } else {
            $child = $context->children()[0];
        }
        $context->forward($child);
        \Swoole\Coroutine::sleep(0.1);
    }
}

```

To execute this, modify the previous `main.php` as follows.

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

function main(): void
{
    run(function () {
        \Swoole\Coroutine\go(function () {
            $system = ActorSystem::create();
            $ref = $system->root()->spawn(
                ActorSystem\Props::fromProducer(
                    fn() => new PlaybackActor()
                )
            );
            $system->root()->send($ref, new PlayMovie('Transformers', 1));
            $system->root()->send($ref, new PlayMovie('Transformers last knight', 2));
            $system->root()->send($ref, new PlayMovie('Transformers age of extinction', 3));
            $system->root()->send($ref, new PlayMovie('Transformers dark of the moon', 4));
            $system->root()->send($ref, new PlayMovie('Transformers revenge of the fallen', 5));
            $system->getLogger()->info('restarting actor');
            $system->root()->send($ref, new Recover());
            $system->root()->send($ref, new PlayMovie('Transformers One', 6));
            $system->root()->send($ref, new PlayMovie('Transformers The Movie', 7));
            $system->getLogger()->info('stopping actor');
            $system->root()->poison($ref);
        });
    });
}

main();
```

Please confirm that when the actor stops, `PlaybackActor stopping` is output to the log.

Also, confirm that the child actor restarting is logged.

![demo1](/images/lifecycle/demo1.png "Demo1")

![demo2](/images/lifecycle/demo2.png "Demo2")
