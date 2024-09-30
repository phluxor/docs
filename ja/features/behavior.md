# Behaviors

Phluxorのアクターはいつでも動作を変更できるため、ステートマシンの実装が簡単に実現できます。

例としてオーディオ プレーヤーをモデル化してみましょう。

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

`Phluxor\ActorSystem\Message\ActorInterface` を実装し、  
アクターの`receive`メソッド内で `Phluxor\ActorSystem\Behavior` クラスの `receive` メソッドを呼び出します。

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

この例で使用されているメッセージオブジェクトは次の3つです。

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

アクターの動作を変更するには、次の3つのメソッドを使用します:

**become**: 現在の動作を渡された動作に置き換え、デフォルトの受信メソッドを置き換えます。

**becomeStacked**: 以前の動作を保持したまま、新しい動作をスタックします。

**unbecomeStacked**: 以前の動作に戻します。

### become

オーディオ プレーヤーの初期状態として、デフォルトの動作を電源オフに設定しましょう。

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

電源オフの状態では、プレーヤーに触れても音楽は再生されません。

「doing nothing（なにもしていません）」というメッセージが出力されます。

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

電源オンの状態では、プレーヤーに触れると音楽が再生されます。

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

再度電源を切ると、以前と同じ「何もしていません」というメッセージが出力されます。

完全な実装例は次のとおりです。

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

エントリポイントとして次のコードを実行します。

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

出力は次のようになります。

![output](/images/behaviors/sample.png "output")

## General Message Handling

現在の動作に関係なく、特定のメッセージを処理したい場合があります。

音楽を聴きながらモッシュするとどうなるか考えてみましょう。

オーディオ プレーヤーが踏まれて壊れる場合があります。

これは、`Phluxor\ActorSystem\Behavior`クラスに動作を委譲する前に、次のような特定のメッセージを処理することで対処できます。

次のメッセージを例として、[モッシュ](https://ja.wikipedia.org/wiki/%E3%83%A2%E3%83%83%E3%82%B7%E3%83%A5)を表現してみましょう。

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

実装は以下のとおりです。

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

動作を確認したいですか？  
完全な実装例は次のとおりです。

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

エントリポイントとして次のコードを実行します。

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

`becomeStacked` メソッドは、以前の動作を保持しながら新しい動作をスタックします。

`unbecome` メソッドは、以前の動作に戻ります。

以下の実装をしてみましょう。
モッシュのあとに `becomeStacked` メソッドを使用して、壊れたオーディオ プレーヤーを置き換えます。  
`unbecome` メソッドを実行すると、壊れたオーディオ プレーヤーに戻ります。 :)

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
