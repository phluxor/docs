# Props

Propsはアクターを作成するために使用される構成オブジェクトです。

`Phluxor\ActorSystem\Props`を利用することでアクター生成のための指定を行うことができます。

## Basic Usage

### fromProducer

`fromProducer()`は、アクターを作成する静的メソッドです。

```php
Props::fromProducer(fn() => new YourActor())
```

`fromProducer()` メソッドは、プロデューサーを使用してPropsオブジェクトを作成します。

最初のパラメーターは、アクターインスタンス (`Closure(): ActorInterface`) を返すクロージャです。

または、`Phluxor\ActorSystem\Message\ProducerInterface` インターフェイスを実装します。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

interface ProducerInterface
{
    /**
     * @return ActorInterface
     */
    public function __invoke(): ActorInterface;
}

```

### fromFunction

`fromFunction()`は`fromProducer()`同様に、アクターを作成する静的メソッドです。

```php
Props::fromFunction(
    new ActorSystem\Message\ReceiveFunction(
        fn(ActorSystem\Context\ContextInterface $context) => $context->message()
    )
);
```

`fromFunction()` メソッドは、関数を含むPropsオブジェクトを作成します。

最初のパラメーターは `Phluxor\ActorSystem\Message\ReceiveFunction` オブジェクトです。
この`ReceiveFunction` は関数をラップするクラスです。

関数は`Phluxor\ActorSystem\Context\ContextInterface` オブジェクトを受け取ります。

## Custom Props

オプションで、カスタムPropsオブジェクトを作成できます。
すべてのオプションは2番目以降の引数で複数指定できます。

いくつか指定可能なカスタムPropsを示します。

### withDispatcher

`withDispatcher()` は、Propsオブジェクトのディスパッチャーを設定するメソッドです。

ディスパッチャーは、アクターにメッセージをディスパッチするために使用されます。

アクターモデルまたはPhluxorについて十分な知識がない限り、カスタマイズはオススメしません。

```php
Props::fromProducer(
    fn() => new YourActor(),
    Props::withDispatcher(new ActorSystem\Dispatcher\SynchronizedDispatcher())
);
```

デフォルトのディスパッチャーは `Phluxor\ActorSystem\Dispatcher\CoroutineDispatcher` クラスを使用します。

このディスパッチャーはSwoole拡張機能用に最適化されています。

デフォルトのディスパッチャーを使用する場合は、設定する必要はありません。

### withMailbox

`withMailbox()` は、Propsオブジェクトのメールボックスを設定するメソッドです。

メールボックスは、アクターのメッセージをキューに入れるために使用されます。

```php
Props::fromProducer(
    fn() => new YourActor(),
    Props::withMailboxProducer(new ActorSystem\Mailbox\Unbounded())
);
```

デフォルトのメールボックスは `Phluxor\ActorSystem\Mailbox\Unbounded` クラスを使用します。

このメールボックスは無制限であり、無制限の数のメッセージを利用できます。

### withSupervisor

`withSupervisor()` は、Propsオブジェクトのスーパーバイザーを設定するメソッドです。

デフォルトの戦略では、10秒のウィンドウ内で故障した子アクターのみを最大10回再起動します。

```php

Props::fromProducer(
    fn() => new VoidActor(),
    ActorSystem\Props::withSupervisor(
        new ActorSystem\Strategy\OneForOneStrategy(
            10,
            new \DateInterval('PT10S'),
            fn() => ActorSystem\Directive::Restart
        )
    )
);
```

デフォルトのスーパーバイザーは `Phluxor\ActorSystem\Strategy\OneForOneStrategy` クラスを使用します。

デフォルトのスーパーバイザーを使用する場合は、設定する必要はありません。

### withReceiverMiddleware

受信ミドルウェアは、アクターがメッセージを受信する前に呼び出して利用できます。

```php

Props::fromProducer(
    fn() => new VoidActor(),
    Props::withReceiverMiddleware(
        $this->mockReceiverMiddleware(
            function (ContextInterface $context, MessageEnvelope $messageEnvelope) {
                if ($messageEnvelope->getMessage() === 'hello') {
                    // logging and other processing
                }
            }
        )
    )
);

// example of mockReceiverMiddleware
private function mockReceiverMiddleware(Closure|ReceiverFunctionInterface $next): Props\ReceiverMiddlewareInterface
{
    return new readonly class($next) implements Props\ReceiverMiddlewareInterface {

        public function __construct(
            private Closure|ReceiverFunctionInterface $next
        ) {
        }

        public function __invoke(
            Closure|ReceiverFunctionInterface $next
        ): ReceiverFunctionInterface {
            return new readonly class($this->next) implements ReceiverFunctionInterface {

                public function __construct(
                    private Closure|ReceiverFunctionInterface $next
                ) {
                }
                
                public function __invoke(
                    ReceiverInterface|ContextInterface $context,
                    MessageEnvelope $messageEnvelope
                ): void {
                    $next = $this->next;
                    $next($context, $messageEnvelope);
                }
            };
        }
    };
}
```

たとえば、メッセージをログに記録したり、その他の処理を実行したりできます。
または、イベントをデータベースに保存する永続化ミドルウェアとしても使用できます。  
CQRS/ESの実装を行う場合は、永続化ミドルウェアを使用することをオススメします。
