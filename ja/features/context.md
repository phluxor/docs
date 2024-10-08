# Context

Phluxorには2つのコンテキストがあります。

`Phluxor\ActorSystem\ActorContext`と`Phluxor\ActorSystem\RootContext`です。

これらの2つのコンテキストは、`Spawner`、`Stopper`、`Infomation`、`Sender`などの共通機能を実装しています。

さらに、**ActorContext**には追加の機能が実装されています。

## Spawner

`Phluxor\ActorSystem\ActorContext`と`Phluxor\ActorSystem\RootContext`には、`spawn`メソッドがあります。

`Spawner`は`Phluxor\ActorSystem\Props`オブジェクトから新しいアクターを作成する機能です。

`Props`オブジェクトは、アクターを作成するための構成オブジェクトです。

```php
<?php

use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

$system = ActorSystem::create();

$spawned = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
```

[Props](/ja/features/props.html) についてもしっかり理解しておきましょう。

## Stopper

`Phluxor\ActorSystem\ActorContext`と`Phluxor\ActorSystem\RootContext`には、`stop`メソッドがあります。

アクターを即座に停止させる、または現在のメールボックスメッセージの処理後に停止を指示する機能です。

`stop`と`poison`にはいくつかのメソッドがありますが、

これらのメソッドには引数として`Phluxor\ActorSystem\Ref`オブジェクトが必要です。

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        $context->stop($context->sender());
        $context->stop($context->self());
        $context->stop(new Ref(new ActorSystem\ProtoBuf\Pid([
            'id' => 'actor name',
        ])));
    }
}
```

## Information

`info`はメソッドではなく、アクターに関する情報を提供する機能です。

`Phluxor\ActorSystem\ActorContext`と`Phluxor\ActorSystem\RootContext`にはアクターの情報に関する機能があります。

現在のアクターの親アクター、現在のアクターの`Phluxor\ActorSystem\Ref`、  
メッセージ送信元のアクターの`Phluxor\ActorSystem\Ref`、  
アクターが存在する`Phluxor\ActorSystem`など、コンテキストの情報を取得できます。

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $msg = $context->message(); // get message
        $context->sender(); // get sender Ref
        $context->self(); // get self Ref
        $contex->parent(); // get parent Ref
        $context->actorSystem(); // get actor system
        $context->logger(); // get logger
        // etc...
    }
}
```

## Sender

`Phluxor\ActorSystem\ActorContext`と`Phluxor\ActorSystem\RootContext`には、`sender`メソッドがあります。

`send`メソッドは、fire-and-forgetスタイルでメッセージを送信し、  
`request`メソッドは、宛先アクターに非同期で応答をリクエストする機能を提供します。

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    // implement receive method
    public function receive(ContextInterface $context): void
    {
        // example - make a actor reference
        $pid = new Ref(new Pid([
            'id' => 'otherActor',
        ]));
        // Of course, 
        // you can also use the spawn method to create an actor reference.
        // fire-and-forget style message
        $context->send($pid, new YourMessage('hello'));
        // request style message
        $context->request($pid, new YourMessage('hello'));
        // future style message
        $context->requestFuture($pid, new YourMessage('hello'), 1);
    }
}
```

## Receiver

`Phluxor\ActorSystem\ActorContext`のみの機能です。

`receive`メソッドは、`Phluxor\ActorSystem\Message\MessageEnvelope`でラップされたメッセージを受信する機能を提供します。

```php
<?php

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\ProtoBuf\Pid
use Phluxor\ActorSystem\Ref;

class YourActor implements ActorInterface
{
    // implement receive method
    public function receive(ContextInterface $context): void
    {
        $context->receive($envelope);
    }
}
```

## Supervisor

`Phluxor\ActorSystem\ActorContext`のみの機能です。

Supervisionは、現在のアクターの監視下にある子アクターのライフサイクルを制御するためのメソッドを提供し、  
階層構造の監視において障害を次のアクターにエスカレーションする機能を提供します。

[supervision](/ja/what/supervision.html)についてもしっかり理解しておきましょう。
