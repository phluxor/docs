# Timeout for receiving messages

By calling the `setReceiveTimeout` method, you can set a timeout for processing before replying to a message.

If the timeout is reached, the actor will receive a `Phluxor\ActorSystem\Message\ReceiveTimeout` message.

This message can be used to trigger a response to the timeout.

```php
public function receive(ContextInterface $context): void
{
    $message = $context->message();
    switch (true) {
        case $message === 'hello':
            $context->setReceiveTimeout(new DateInterval('PT1S'));
            // any process that takes more than 1 second will be terminated
            break;
        case $message instanceof ActorSystem\Message\ReceiveTimeout:
            // receive timeout message
            break;
    }
}
```

## Note

that this timeout may insert a ReceiveTimeout message into the mailbox immediately after the message reaches the mailbox.

Once set, the receive timeout remains active.

This means it will continue to occur repeatedly after periods of inactivity.

To turn off the timeout, pass `new \DateInterval('PT0S')` to `setReceiveTimeout`.

```php
public function receive(ContextInterface $context): void
{
    $message = $context->message();
    switch (true) {
        case $message === 'hello':
            $context->setReceiveTimeout(new DateInterval('PT1S'));
            // any process that takes more than 1 second will be terminated
            break;
        case $message === 'reset':
            // reset the timeout
            $context->setReceiveTimeout(new DateInterval('PT0S'));
            break;
        case $message instanceof ActorSystem\Message\ReceiveTimeout:
            // no timeout
            break;
    }
}
```

## NoInfluence

There is a way to send a message to an actor without resetting the receive timeout timer for messages that have no impact.

To do this, implement the `Phluxor\ActorSystem\Message\NotInfluenceReceiveTimeoutInterface` in the message.

example:

```php
<?php

declare(strict_types=1);

namespace Acme\Message;

use Phluxor\ActorSystem\Message\NotInfluenceReceiveTimeoutInterface;

class NoInfluence implements NotInfluenceReceiveTimeoutInterface
{
}
```
