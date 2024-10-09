# Middleware

Middleware is a feature for adding hooks before and after processing an actor's messages.

You can add middleware to both received and sent messages.  
In Phluxor, these are called ReceiverMiddleware and SenderMiddleware,  
and you can add middleware using `Phluxor\ActorSystem\Props`.

## ReceiverMiddleware

Middleware for received messages adds hooks before and after an actor receives a message.

You can add middleware to an actor's received messages using the `Props::withReceiverMiddleware` method.

You can add any number of middlewares, which will be executed in the order they were added.

The `Props::withReceiverMiddleware` method requires a class that implements `Phluxor\ActorSystem\Props\ReceiverMiddlewareInterface`,

and that class must implement an `__invoke` method, which returns `Phluxor\ActorSystem\Message\ReceiverFunctionInterface`.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Props;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;

interface ReceiverMiddlewareInterface
{
    /**
     * @param Closure(ReceiverInterface|ContextInterface, MessageEnvelope): void|ReceiverFunctionInterface $next
     * @return ReceiverFunctionInterface
     */
    public function __invoke(
        Closure|ReceiverFunctionInterface $next
    ): ReceiverFunctionInterface;
}
```

ReceiverFunctionInterface is a function that is executed when an actor receives a message.

Use the `$next` argument to call the next middleware.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;

interface ReceiverFunctionInterface
{
    /**
     * @param ReceiverInterface|ContextInterface $context
     * @param MessageEnvelope $messageEnvelope
     * @return void
     */
    public function __invoke(
        ReceiverInterface|ContextInterface $context,
        MessageEnvelope $messageEnvelope
    ): void;
}
```

`ReceiverFunctionInterface` is a function that is executed when an actor receives a message.

`$context` represents the actor's context, and `$messageEnvelope` represents the received message.

This allows you to customize the processing for messages received by the actor.

An implementation example is as follows:

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;

readonly class ReceiverFactory implements ReceiverFunctionInterface
{
    public function __construct(
        private Closure|ReceiverFunctionInterface $next
    ) {
    }

    public function __invoke(
        ContextInterface|ReceiverInterface $context,
        MessageEnvelope $messageEnvelope
    ): void {
        var_dump('before receiver');
        $next = $this->next;
        $next($context, $messageEnvelope); // next middleware or actor receive
        var_dump('after receiver');
    }
}

```

In the example above, logs are output before and after the actor receives a message.

The class for adding middleware would look like this:

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;
use Phluxor\ActorSystem\Props\ReceiverMiddlewareInterface;

class ReceiverMiddleware implements ReceiverMiddlewareInterface
{
    public function __invoke(ReceiverFunctionInterface|Closure $next): ReceiverFunctionInterface
    {
        return new ReceiverFactory($next);
    }
}

```

To apply these to an actor, use the `Props::withReceiverMiddleware` method.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Example\Middleware\ReceiverMiddleware;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new ChildActor(),
                        Props::withReceiverMiddleware(
                            new ReceiverMiddleware()
                        )
                    )
                );
                $context->send($ref, $message);
                break;
        }
    }
}

```

When multiple middlewares are added, they are executed as shown below.

```bash
string(15) "before receiver" // first receiver middleware
string(16) "before receiver2" // second receiver middleware
string(15) "after receiver2" // second receiver middleware
string(14) "after receiver" // first receiver middleware
```

## SenderMiddleware

Middleware for sent messages adds hooks before and after an actor sends a message.

You can add middleware to an actor's sent messages using the `Props::withSenderMiddleware` method.

As with `ReceiverMiddleware`, you can add any number of middlewares,  
and they will be executed in the order they were added.

The `Props::withSenderMiddleware` method requires a class that implements `Phluxor\ActorSystem\Props\SenderMiddlewareInterface`,

and that class must implement an `__invoke` method that returns `Phluxor\ActorSystem\Message\SenderFunctionInterface`.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Props;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Ref;

interface SenderMiddlewareInterface
{
    /**
     * @param Closure(SenderInterface|ContextInterface, Ref, MessageEnvelope): void|SenderFunctionInterface $next
     * @return SenderFunctionInterface
     */
    public function __invoke(Closure|SenderFunctionInterface $next): SenderFunctionInterface;
}
```

`SenderFunctionInterface` is a function that is executed when an actor sends a message.

Use the `$next` argument to call the next middleware.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Ref;

interface SenderFunctionInterface
{
    /**
     * @param SenderInterface $context
     * @param Ref|null $target
     * @param MessageEnvelope $messageEnvelope
     * @return void
     */
    public function __invoke(
        SenderInterface $context,
        Ref|null $target,
        MessageEnvelope $messageEnvelope
    ): void;
}

```

`SenderFunctionInterface` is a function that is executed when an actor sends a message.

`$context` represents the actor's context, `$target` represents the target actor,

and `$messageEnvelope` represents the message being sent.

This allows you to customize the processing for messages sent by the actor.

The basic implementation method is similar to that of middleware for received messages.

An example is shown below:

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Ref;

readonly class SenderFactory implements SenderFunctionInterface
{
    public function __construct(
        private SenderFunctionInterface|Closure $next
    ) {
    }

    public function __invoke(SenderInterface $context, ?Ref $target, MessageEnvelope $messageEnvelope): void
    {
        var_dump('before sender');
        $next = $this->next;
        $next($context, $target, $messageEnvelope);
        var_dump('after sender');
    }
}

```

The class for adding middleware would look like this:

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Props\SenderMiddlewareInterface;

class SenderMiddleware implements SenderMiddlewareInterface
{
    public function __invoke(SenderFunctionInterface|Closure $next): SenderFunctionInterface
    {
        return new SenderFactory($next);
    }
}

```

To apply these to an actor, use the `Props::withSenderMiddleware` method.

```php
<?php

use Phluxor\ActorSystem;

$system = ActorSystem::create();
$ref = $system->root()->spawn(
    ActorSystem\Props::fromProducer(
        fn() => new Example\ParentActor(),
        ActorSystem\Props::withSenderMiddleware(
            new Example\Middleware\SenderMiddleware()
        )
    )
);
```

The usage within the actor context is the same.

Of course, you can also add multiple middlewares.

### Example of Middleware Usage

Middleware is a feature for adding hooks before and after processing an actor's messages,

making it useful for implementing Event Sourcing or CQRS as well.

For example, you can implement message persistence as middleware that issues events before and after an actor receives a message.

If you would like to learn more about these mechanisms,  
please refer to the sections on persistence and event sourcing (documentation in preparation).

## MailboxMiddleware

MailboxMiddleware is a feature that allows you to add hooks when an actor's message arrives in the mailbox.

Similar to other middleware, MailboxMiddleware can be added using `Phluxor\ActorSystem\Props`.

To use MailboxMiddleware,  
you need to provide a class that implements `Phluxor\ActorSystem\Mailbox\MailboxMiddlewareInterface`,

and use the `Props::withMailboxProducer` method to add the MailboxMiddleware.

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

interface MailboxMiddlewareInterface
{
    public function mailboxStared(): void;

    public function messagePosted(mixed $message): void;

    public function messageReceived(mixed $message): void;

    public function mailboxEmpty(): void;
}

```

The `mailboxStarted` method is called when the Mailbox itself starts.

The `messagePosted` method is called when a system message or user message is sent to the Mailbox.

After being called, the message is pushed to the Mailbox.

The `messageReceived` method is called when a system message or user message reaches or is received by the Mailbox.

The `mailboxEmpty` method is called each time the Mailbox becomes empty.

Customizing the Mailbox requires a basic understanding of Phluxor,  
so unless you have a solid understanding of the actor model and Phluxor,  
customization is not recommended.

However, adding middleware is straightforward.

By default, Phluxor uses an **Unbounded Mailbox**.
To add middleware to this Mailbox, you can do so as follows:

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Example\Middleware\MailboxMiddleware;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Mailbox\Unbounded;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new ChildActor(),
                        Props::withMailboxProducer(
                            new Unbounded(new MailboxMiddleware())
                        )
                    )
                );
                $context->send($ref, $message);
                break;
        }
    }
}

```
