# Mailbox

Actors can interact only through messages.

These messages are not sent directly to the actor but are sent to the mailbox that each actor possesses.  
When the actor becomes ready to process them,  
the messages are pushed to the actor.

The default mailbox consists of two message queues: system messages and user messages.  
System messages are used internally by the system—for example,  
to suspend or resume the mailbox in case of a crash—and are also used for managing each actor (starting, stopping, restarting, etc.).  
User messages can be freely used by developers.  
These messages can be accessed from the actor context.

Messages in the mailbox are generally delivered in FIFO order,  
but system messages are given priority and processed before user messages.

The following rules apply to an actor's mailbox:

- Sending messages can be performed simultaneously by multiple producers (MPSC: multi-producer, single-consumer).

- Receiving messages is done sequentially for each actor. Different rules can optionally be applied to special mailboxes like system messages.

- Mailboxes cannot be shared between actors. They are not shared internally either.

Since the default mailbox is unlimited,  
you can specify an arbitrary enqueue limit on the mailbox to adjust throughput and other factors.

## Changing the Mailbox

To use a specific mailbox implementation, you can customize the `Phluxor\ActorSystem\Props`.  
you can also use the `Phluxor\ActorSystem\Props::withMailboxProducer` method.

```php
Props::fromProducer(fn() => new YourActor(),
    Props::withMailboxProducer(YourMailboxProducer)
);
```

you can also implement the `Phluxor\ActorSystem\Mailbox\MailboxProducerInterface` interface.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

interface MailboxProducerInterface
{
    /**
     * @return MailboxInterface
     */
    public function __invoke(): MailboxInterface;
}
```

Mailbox implementations must implement the `Phluxor\ActorSystem\Mailbox\MailboxInterface` interface.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

use Phluxor\ActorSystem\Dispatcher\DispatcherInterface;

interface MailboxInterface
{
    /**
     * @param mixed $message
     * @return void
     */
    public function postUserMessage(mixed $message): void;

    /**
     * @param mixed $message
     * @return void
     */
    public function postSystemMessage(mixed $message): void;

    /**
     * @return void
     */
    public function start(): void;

    /**
     * @return int
     */
    public function userMessageCount(): int;

    /**
     * @param MessageInvokerInterface $invoker
     * @param DispatcherInterface $dispatcher
     * @return void
     */
    public function registerHandlers(
        MessageInvokerInterface $invoker,
        DispatcherInterface $dispatcher
    ): void;
}
```

## Unbounded Mailbox

The unbounded mailbox is a convenient default option available for use.  

However, if a large number of messages are added to the mailbox faster than the actor can process them,  
it may lead to memory shortages and potential instability.  
For this reason, you can specify a bounded mailbox.  

When using a The bounded mailbox,  
if the mailbox becomes full, new messages are forwarded to the dead letter queue.
