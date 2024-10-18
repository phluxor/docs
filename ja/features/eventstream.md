# EventStream

Phluxorでは`EventStream`という機能を提供しています。

これはアクターがイベントをPublishし、他のアクターがそれらのイベントをサブスクライブできる組み込み機能となります。  

各アクターのコンテキストから `getEventStream()` を呼び出すことで、**EventStream** にアクセスできます。  
（ActorSystemのルートからは `getEventStream()` を直接呼び出せます。）

**EventStream** は、  
アクターシステム内の任意のアクターが任意のメッセージをサブスクライブできるようにするためのものです。

もちろん、一般的なPubSubシステムと同様に、複数のアクターが同時にサブスクライブすることもできます。

サブスクライブするには、EventStreamクラスの `subscribe` メソッドを使用し、  
コールバックを通じてメッセージを受信します。

![flow](/images/eventstream/eventstream.png "eventstream")

```php
<?php

declare(strict_types=1);

namespace Example;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;

class SubscribeTwoActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $subscription = $context->actorSystem()->getEventStream()->subscribe(
            function ($message) use ($context) {
                $context->logger()->info('Received message', ['message' => $message]);
            }
        );
    }
}

```

サブスクリプションが不要になった場合は、  
EventStreamクラスの `unsubscribe` メソッドを使用してサブスクリプションを削除します。

```php
$subscription = $context->actorSystem()->getEventStream()->subscribe(
    function ($message) use ($context) {
        $context->logger()->info('Received message', ['message' => $message]);
    }
);
$context->actorSystem()->getEventStream()->unsubscribe($subscription);
```

メッセージはどのような種類でも構いません。  
たとえば以下のようなシンプルなメッセージや、  
アクターコンテキストからアクセスできるメッセージも使用できます。

```php
<?php

declare(strict_types=1);

namespace Example\Message;

readonly class Ordered
{
    public function __construct(
        public string $dish
    ) {}
}
```

EventStreamでメッセージを公開するのは簡単です。　　
`publish` メソッドを呼び出すだけです。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Ordered;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;

class PublishActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        if ($message === 'order') {
            $context->actorSystem()->getEventStream()->publish(
                new Ordered('Pizza')
            );
        }
    }
}
```

すぐに動作を確認したい場合は、次のコードで試してみてください。

```php
<?php

declare(strict_types=1);

use Example\PublishActor;
use Example\SubscribeActor;
use Example\SubscribeTwoActor;
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();

        $ref = $system->root()->spawn(Props::fromProducer(fn() => new PublishActor()));
        $system->root()->spawn(Props::fromProducer(fn() => new SubscribeActor()));
        $system->root()->spawn(Props::fromProducer(fn() => new SubscribeTwoActor()));
        $system->root()->send($ref, 'order');
    });
});
```
