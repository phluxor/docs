# Behaviors

Phluxor actors can change their behavior at any time, enabling the implementation of a State Machine.

Let's model the audio player as an example.

```php
<?php

declare(strict_types=1);

namespace Example;

use Phluxor\ActorSystem\Behavior;
use Phluxor\ActorSystem\Message\ActorInterface;

class AudioPlayer implements ActorInterface
{
    private Behavior $behavior;

    public function __construct()
    {
        $this->behavior = new Behavior();
    }
}
```

Implement the `Phluxor\ActorSystem\Message\ActorInterface` and call the `receive` method of the `Phluxor\ActorSystem\Behavior` class inside the actor's receive method.

```php
<?php

declare(strict_types=1);

namespace Example;

use Phluxor\ActorSystem\Behavior;
use Phluxor\ActorSystem\Message\ActorInterface;

class AudioPlayer implements ActorInterface
{
    private Behavior $behavior;

    public function __construct()
    {
        $this->behavior = new Behavior();
    }

    public function receive(ContextInterface $context): void
    {
        $this->behavior->receive($context);
    }
|
```

The message objects used in this example are the following three

```php
<?php

declare(strict_types=1);

namespace Example\Message;

class PowerOn
{
}

```

```php
<?php

declare(strict_types=1);

namespace Example\Message;

class PowerOff
{
}

```

```php
<?php

declare(strict_types=1);

namespace Example\Message;

class Touch
{
}

```

## Changing behaviors

To change an actor's behavior, three methods are used:

**become**: Replaces the current behavior with the one passed in, replacing the default receive method.

**becomeStacked**: Stacks the new behavior while retaining the previous one.

**unbecomeStacked**: Reverts to the previous behavior.

### become

As the initial state of the audio player, let's set the default behavior to Power Off.

```php
<?php

declare(strict_types=1);

namespace Example;

use Phluxor\ActorSystem\Behavior;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\ReceiveFunction;

class AudioPlayer implements ActorInterface
{
    private Behavior $behavior;

    public function __construct()
    {
        $this->behavior = new Behavior();
        $this->behavior->become(
            new ReceiveFunction(
                fn(ContextInterface $context) => $this->off($context)
            )
        );
    }
}
```

When in the Power Off state, touching the player will not play music.

A message saying "doing nothing" will be output.

```php
    private function off(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
                $context->logger()->info('Powering on');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->on($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('doing nothing');
                break;
        }
    }
```

When in the Power On state, touching the player will play music.

```php
    private function on(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOff:
                $context->logger()->info('Powering off');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('starting to play');
                break;
        }
    }
```

If the power is turned off again, the same "doing nothing" message will be output as before.

The complete implementation example is as follows.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\PowerOff;
use Example\Message\PowerOn;
use Example\Message\Touch;
use Phluxor\ActorSystem\Behavior;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\ReceiveFunction;

class AudioPlayer implements ActorInterface
{
    private Behavior $behavior;

    public function __construct()
    {
        $this->behavior = new Behavior();
        $this->behavior->become(
            new ReceiveFunction(
                fn(ContextInterface $context) => $this->off($context)
            )
        );
    }

    public function receive(ContextInterface $context): void
    {
        $this->behavior->receive($context);
    }

    private function off(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
                $context->logger()->info('Powering on');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->on($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('doing nothing');
                break;
        }
    }

    private function on(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOff:
                $context->logger()->info('Powering off');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('starting to play');
                break;
        }
    }
}
```

Run the following code as the entry point.

```php
<?php

declare(strict_types=1);

use Example\AudioPlayer;
use Example\Message\PowerOff;
use Example\Message\PowerOn;
use Example\Message\Touch;
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();
        $ref = $system->root()->spawn(
            Props::fromProducer(fn() => new AudioPlayer())
        );
        $system->root()->send($ref, new Touch());
        $system->root()->send($ref, new PowerOn());
        $system->root()->send($ref, new Touch());
        $system->root()->send($ref, new PowerOff());
        $system->root()->send($ref, new Touch());
    });
});

```

The output will be as follows.

![output](/images/behaviors/sample.png "output")

## General Message Handling

There may be cases where you want to process certain messages regardless of the current behavior.

Let's consider what happens if you mosh while listening to music.

The audio player might get stepped on and break.

This can be handled by processing specific messages like the following before delegating to the `Phluxor\ActorSystem\Behavior` class.

Let's represent mosh using the following messages as an example.

```php
<?php

declare(strict_types=1);

namespace Example\Message;

class Moshing
{
}

```

```php
<?php

declare(strict_types=1);

namespace Example\Message;

class ReplaceAudioPlayer
{
}

```

The implementation is as follows.

```php
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        if ($message instanceof Moshing) {
            $context->respond('moshing');
            $this->behavior->become(
                new ReceiveFunction(fn(ContextInterface $context) => $this->moshed($context))
            );
        }
        $this->behavior->receive($context);
    }

    private function moshed(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
            case $message instanceof PowerOff:
                $context->logger()->info('broken');
                break;
            case $message instanceof Touch:
                $context->logger()->info('broken!!!');
                break;
            case $message instanceof ReplaceAudioPlayer:
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                break;
        }
    }
```

The complete implementation example is as follows.

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Moshing;
use Example\Message\PowerOff;
use Example\Message\PowerOn;
use Example\Message\ReplaceAudioPlayer;
use Example\Message\Touch;
use Phluxor\ActorSystem\Behavior;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\ReceiveFunction;

class AudioPlayer implements ActorInterface
{
    private Behavior $behavior;

    public function __construct()
    {
        $this->behavior = new Behavior();
        $this->behavior->become(
            new ReceiveFunction(
                fn(ContextInterface $context) => $this->off($context)
            )
        );
    }

    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        if ($message instanceof Moshing) {
            $context->respond('moshing');
            $this->behavior->become(
                new ReceiveFunction(fn(ContextInterface $context) => $this->moshed($context))
            );
        }
        $this->behavior->receive($context);
    }

    private function off(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
                $context->logger()->info('Powering on');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->on($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('doing nothing');
                break;
        }
    }

    private function on(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOff:
                $context->logger()->info('Powering off');
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                break;
            case $message instanceof Touch:
                $context->logger()->info('starting to play');
                break;
        }
    }

    private function moshed(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
            case $message instanceof PowerOff:
                $context->logger()->info('broken');
                break;
            case $message instanceof Touch:
                $context->logger()->info('broken!!!');
                break;
            case $message instanceof ReplaceAudioPlayer:
                $this->behavior->become(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                break;
        }
    }
}
```

Run the following code as the entry point.

```php
<?php

declare(strict_types=1);

use Example\AudioPlayer;
use Example\Message\Moshing;
use Example\Message\PowerOff;
use Example\Message\PowerOn;
use Example\Message\ReplaceAudioPlayer;
use Example\Message\Touch;
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();
        $ref = $system->root()->spawn(
            Props::fromProducer(fn() => new AudioPlayer())
        );
        $system->root()->send($ref, new Touch());
        $system->root()->send($ref, new PowerOn());
        $system->root()->send($ref, new Touch());
        $system->root()->send($ref, new PowerOff());
        $future = $system->root()->requestFuture($ref, new Moshing(), 1);
        $system->getLogger()->info($future->result()->value());
        $system->root()->send($ref, new Touch());
        $system->root()->send($ref, new PowerOn());
        $system->root()->send($ref, new ReplaceAudioPlayer());
        $system->root()->send($ref, new PowerOn());
        $system->root()->send($ref, new Touch());
    });
});
```

## BecomeStacked / UnbecomeStacked

The `becomeStacked` method stacks the new behavior while retaining the previous one.

The `unbecome` method reverts to the previous behavior.

Let's implement it as follows:

After moshing, use the `becomeStacked` method to replace the broken audio player.

Execute the `unbecome` method to go back to the broken audio player. :)

```php
    private function moshed(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof PowerOn:
            case $message instanceof PowerOff:
                $context->logger()->info('broken');
                break;
            case $message instanceof Touch:
                $context->logger()->info('broken!!!');
                break;
            case $message instanceof ReplaceAudioPlayer:
                $this->behavior->becomeStacked(
                    new ReceiveFunction(fn(ContextInterface $context) => $this->off($context))
                );
                $this->behavior->unbecome();
                break;
        }
    }
```

![output](/images/behaviors/sample2.png "output")
