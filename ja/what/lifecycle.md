# Actor Lifecycle / アクターライフサイクル

Phluxorのアクターライフサイクルは、アクターの生成から終了までの一連の流れを表します。  

基本的なケースとしては、  
生成されたアクターは最初に具現化され、  
Startedメッセージが送信されます、  
そのアクターはアプリケーションがシャットダウンするまで生存し続けます。  

## アクターの停止

アクターを停止するには、さまざまな方法があります：

コンテキストから **Stop** メッセージを送信する。

`Phluxor\ActorSystem\ActorContext`インスタンスの **stop** メソッドを呼び出す。

## Failure and supervision

アクターがメッセージの処理に失敗した場合、  
アクターのメールボックスは一時的に停止され、処理が中断されます。

この失敗は他のアクター、もしくは失敗したアクターを監督している親アクターにエスカレートされます（[supervision](/en/what/supervision.html)を参照）。

監督者が決定した指示に応じて、  
アクターは自動的に再開、再起動、または停止する可能性があります。

### Resume

メールボックスは処理を再開し、  
アクターは以前に失敗したメッセージから実行を続けます。

### Stop

アクターが停止されると、  
アクターとして機能していたオブジェクトは破棄されます。

停止前にアクターは**Stop**メッセージを受信し、停止後にも**Stop**メッセージを受信します。

### Restart

アクターに**Restart**メッセージが送信され、再起動が試みられていることが通知されます。

この処理が行われた後、アクターは停止しアクターとして機能していたオブジェクトは破棄され、再作成されます。

その後、メールボックスは再開され、新しく作成されたアクターが**Start**メッセージを受信します。

再起動後に過去のメッセージは取得できません。

## Handling events

`Phluxor\ActorSystem\Message\Started`は、アクターが生成または再起動された後に最初に受信するメッセージです。

データベースからのデータのロードなど、  
アクターの初期状態を設定する必要がある場合はこのメッセージを処理する必要があります。

`Phluxor\ActorSystem\Message\Restarting`はアクターが再起動される直前に送信され、  
`Phluxor\ActorSystem\Message\Stopping`はアクターが停止される直前に送信されます。

いずれの場合もアクターのオブジェクトは破棄されるため、

適切なシャットダウンを確実に行うためのロジック（例: 状態をデータベースに保存する、メッセージングシステムにメッセージを送信するなど）  
を実行する必要がある場合は、これらのメッセージを処理する必要があります。

`Phluxor\ActorSystem\Message\Stopped`はアクターが停止したときに送信され、  
アクターおよびその関連オブジェクトはシステムから切り離されます。

この段階ではアクターはメッセージの送受信ができなくなり、  
このメッセージが処理された後、オブジェクトは完全に破棄されます。

## Flow

アクターのライフサイクルの流れは次のとおりです：

![flow](/images/lifecycle/lifecycle_flow.png "Flows")

## Example

シンプルなアプリケーションを通じて、アクターのライフサイクルを素早く理解しましょう。

[Sample code](https://github.com/ytake/phluxor-example/tree/main/lifecycle)

まず、映画の再生をリクエストするメッセージを作成します。

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

このメッセージは映画の再生をリクエストするために使用されます。

```php
$system->root()->send($ref, new PlayMovie('Transformers', 1));
$system->root()->send($ref, new PlayMovie('Transformers last knight', 2));
$system->root()->send($ref, new PlayMovie('Transformers age of extinction', 3));
$system->root()->send($ref, new PlayMovie('Transformers dark of the moon', 4));
$system->root()->send($ref, new PlayMovie('Transformers revenge of the fallen', 5));
```

次に、映画を再生するアクターを作成します。

このアクターは、PlayMovieメッセージを受信すると映画を再生します。

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

このアクターを実行するには、アクターシステムに登録して起動する必要があります。

次のように`main.php`ファイルを作成して実行してみましょう。

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

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
```

映画のタイトルとユーザーIDがログに出力されていることを確認してください。

ここからアクターのライフサイクルを理解するために、内部メッセージに応答するように修正してみましょう。

## Started

`Phluxor\ActorSystem\Message\Started`は、アクターが作成された後に最初に送信されるメッセージです。

このメッセージを処理する処理を追加しましょう。

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

これを実行するとアクターが作成されたときに`PlaybackActor started`がログに出力されることを確認できます。

## Restarting

アクターで障害が発生すると、  
アクターシステムは問題を修正するためにアクターを再起動し、  
再起動が行われることを通知するために `Phluxor\ActorSystem\Message\Restarting` メッセージをアクターに送信します。

`Started`メッセージとは異なり、`Restarting`メッセージの処理は少し異なります。

ここで `PhluxorExample\Message\Recover` メッセージを追加し、  
子アクターがこのメッセージを受信するとクラッシュし障害を検知して再起動するようにします。

それでは `PhluxorExample\Message\Recover` メッセージを作成しましょう。

```php
<?php

declare(strict_types=1);

namespace PhluxorExample\Message;

class Recover
{
}
```

次に、`Recover`メッセージを受信するとクラッシュする`ChildActor`を作成します。

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

次に `PlaybackActor` が `ChildActor` を作成し、  
`Recover` メッセージを子アクターに転送するようにします。

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

`recoverMessageHandler` メソッドでは、親アクターが子アクターを作成し、`Recover` メッセージを子アクターに転送します。

子アクターがこのメッセージを受信してクラッシュした後、  
再起動後に `Phluxor\ActorSystem\Message\Restarting` メッセージを受信し、  
再起動したことをログに記録します。

これを実行するために、以前の`main.php`を次のように修正します。

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

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
```

子アクターがクラッシュして再起動したこと、そしてそれがログに記録されたことを確認してください。

確認できましたか？

## Stopping

アクターが停止する直前に、アクターシステムは`Phluxor\ActorSystem\Message\Stopping`メッセージを送信します。

**Stopping**はリソースの解放、クリーンアップ、外部リソースからの切断などに使用されます。

これを実装するために、`Phluxor\ActorSystem\Message\Stopping`メッセージを処理する処理を追加します。

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

これを実行するために、以前の`main.php`を次のように修正します。

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

use Phluxor\ActorSystem;

use PhluxorExample\Message\PlayMovie;
use PhluxorExample\Message\Recover;
use PhluxorExample\PlaybackActor;

use function Swoole\Coroutine\run;

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
```

アクターが停止すると、`PlaybackActor stopping`がログに出力されることを確認してください。

また、子アクターの再起動がログに記録されていることも確認してください。

![demo1](/images/lifecycle/lifecycle_demo1.png "Demo1")

![demo2](/images/lifecycle/lifecycle_demo2.png "Demo2")
