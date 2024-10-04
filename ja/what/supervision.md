# Supervision

監督は親アクターが子アクターの状態を監督する機能で、  
監視はアクターモデルをサポートするツールキットの特徴のひとつで、アクターが他のアクターの状態を監視する機能です。

一般的なWebアプリケーションフレームワークや、一般的な処理フローからは想像しにくいかもしれませんが、  
とあるアクターが異常終了した際に監視しているアクターに異常終了自体を通知したり、  
親アクターが子アクターの再起動方法を指定できます。

これまで簡単にヒエラルキーについて解説しましたが、このヒエラルキーと監督は関連しており、  
しっかり把握するのがアクターモデルを理解するための一歩となります。

そんな監督・監視の概念と、Phluxorにおいて監視とはどのような意味を持つかについて説明します。

## What Supervision Means

親アクターは子アクターにタスクを委任することができると同時に、  
その子アクターの失敗に対応する責務を持っています（監督者になる）。  
（Akka、Pekko、Proto Actorなども同様に監督・監視機能を持っています）

*ここで指す子アクターとは、自身で生成 (spawn) した処理を移譲するアクターすべてを指します。

子アクターの失敗を検知すると（PhluxorではExceptionを検知します）、  
自分自身とすべての子アクターを一時停止し、失敗を通知するメッセージを送信します。

これには4つのオプションがあります。

Resume : アクターを再開させ、蓄積された内部状態を保持  
Restart : アクターを再起動し、蓄積された内部状態をクリア  
Stop : アクターを永久に停止し、メッセージ処理を行わない  
Escalate : 階層内における自アクターに失敗を親アクターにエスカレートする  

アクターモデルにおけるアクターは常にヒエラルキーの一部として考えなければなりません。

失敗に合わせてどのように子アクターに指示を出すのかを実装しながら考えておきましょう。

とりあえず再起動させてしまえ！でも構いません。

伝統的なPHPアプリケーションのようにExceptionをキャッチして処理するのではなく、  
アクターモデルでは失敗を受け取り、それに対応する戦略を決定することが重要です。

ではどんなことに気をつけておくべきなのか、少し説明します。  
下記の動作は、ヒエラルキー内のアクターを操作するときに頭に入れておく必要があります。

監督はExceptionを検知すると上記の4つの選択肢のいずれかに変換できるように構成されています。

特定のアクターにのみ異なる戦略を適用したいと思うこともあるかもしれませんが、  
アクターを作成するときに監督についての戦略（スーパーバイザー戦略といいます）を設定できるため、
アクター毎に異なる戦略を持つアクターを事前に作成できる、ということになります。

なおPhluxorでは「親の監督」のみがサポートされています。

アクターは他のアクター（親）によってのみ作成でき、トップレベルのアクターはアクターシステムによって提供されます。

このため親アクターは子アクターの監督者としての役割を果たします。

アクターが外部から監督されることはない、と保証されていますので、  
意図しない別なところから不意にエラーが捕捉されることはない、としっかり理解しておきましょう。

親アクターが子アクターの停止を検知する例を以下に示します。

停止するアクターは以下の例のようになります。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                $context->stop($context->self());
                break;
        }
    }
}
```

停止を受けとる親アクターは以下の例のようになります。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;
use Phluxor\ActorSystem\ProtoBuf\Terminated;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(Props::fromProducer(fn() => new ChildActor()));
                $context->send($ref, $message);
                break;
            case $message instanceof Terminated:
                $context->logger()->info('terminated', [
                    'who' => $message->getWho()->getId(),
                    'why' => $message->getWhy(),
                ]);
                break;
        }
    }
}

```

子アクターが停止すると、親アクターは`Terminated`メッセージを受信し、  
コンソールには次のように表示されます。

![watch](/images/supervision/parent_watch.png "watch")

Exceptionで停止した場合は、下記のように検知されリカバリーが行われます。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                // throw exception!
                throw new \Exception('hi, I am an exception');
                break;
        }
    }
}

```

![recover](/images/supervision/actor_recover.png "recover")

## What Lifecycle Monitoring Means

これまで説明してきたようにアクターはこれまでの伝統的なアプリケーション、  
もしくはオブジェクトというよりも有機的ななにかに見えて来ると思います）。

アクターのライフサイクルは、アクターが生成された時点から終了するまでの一連の流れを表します。

まず、アクターは生成時に完全に生きている状態となります。  
そしてアクターは停止などをすることによって生存していない、死んだ状態となります。  
これの状態を理解できるのは監督者に当たる親以外は通常わかりません。

しかしこの「生きている状態」から「死んでいる状態」への変化は監視で利用できます。  
監督は何かしらの障害に反応しますが、監視はこの「死んだ状態」、つまりアクターの終了に反応するために利用できます。

「生きている状態」から「死んでいる状態」への変化をライフサイクルとしてこれを監視するのがライフサイクルモニタリングです。

ライフサイクル監視は監視アクターが受け取る`Phluxor\ActorSystem\ProtoBuf\Terminated`メッセージを使用して実装できます。

`Phluxor\ActorSystem\ProtoBuf\Terminated` メッセージを受信するために監視を開始しなければなりませんが、  
これは`$context->watch($targetRef)` をコールすることで開始できます。

最後に監視の停止ですが、`$context->unwatch($targetRef)`をコールするだけです。  
重要なのは監視リクエストと対象の終了がどの順序で発生してもメッセージが配信されることです。  
つまり、登録時に対象が停止状態にあってもメッセージを受け取ることができます。

監督者に当たるアクターが子アクターを単に再起動できない場合、  
たとえばアクターの初期化中にエラーが発生したときなどに、子アクターを終了させる必要があります。  
そんなとき、監視はとくに役立ちます。

監督者はその子アクターを監視し、終了したら再作成するか、あとで再試行するようにスケジュールできます。

例として、Webサーバーアクターがあるとしましょう。

Webサーバーアクターは複数の子アクターを使ってリクエストを処理しますが、  
初期化時にデータベース接続が失敗した場合、子アクターを再起動しても問題は解決しません。

この場合、監督者は子アクターの終了を監視しデータベース接続の再試行をスケジュールできます。  
もうひとつの一般的な使用例は、アクターが外部リソースの不在時に失敗する必要がある場合です。

この外部リソースはそのアクター自身の子であることもあります。  
たとえば、監督者アクターがデータベース接続アクターを持っていて、  
第三者が`$context->stop($ref)`メソッドや`$context->poison($ref)`を使ってその子アクターを終了させると、  
監督者もその影響を受ける可能性があります。

そんなときに利用できるでしょう。

## Supervisor Strategy

これまで監視に関連したものを解説しましたが、  
外部リソースなどの影響でアクターが障害時にどのように再起動させていくかを実装できます。

たとえばデータベースなどの高負荷状態にあるものに影響してアクターがクラッシュしてしまう場合、  
少しずつ時間を空けて再起動と再接続を行うような戦略などがあります。

### ExponentialBackoffStrategy

`ExponentialBackoffStrategy`は、指数バックオフ戦略を実装するための戦略です。  
この戦略は、アクターが再起動されるたびに指数関数的に遅延を増やします。

```php
new Phluxor\ActorSystem\Strategy\ExponentialBackoffStrategy(
    new DateInterval("PT02S"),
    new DateInterval("PT01S")
);
```

第1引数はbackoffWindow (DateInterval) となっており、  
どのような時間枠で再起動を行うかを指定します。

第2引数はinitialBackoff (DateInterval) で、最初に再起動を開始する時間指定です。

実際の利用例は以下の通りです。

簡単に再起動が確認できるように子アクター生成時に例外をスローする例を示します。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Message\Restarting;

class ChildActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Started:
                throw new \Exception('hi, I am an exception');
            case $message instanceof Restarting:
                $context->logger()->info('restarting...');
                break;
            case $message instanceof Hello:
                $context->logger()->info('Hello ' . $message->name);
                break;
        }
    }
}
```

次に親アクターに対してExponentialBackoffStrategy戦略利用を指定します。

```php
<?php

declare(strict_types=1);

use Phluxor\ActorSystem;

use function Swoole\Coroutine\run;

require_once __DIR__ . '/vendor/autoload.php';

run(function () {
    \Swoole\Coroutine\go(function () {
        $system = ActorSystem::create();
        $ref = $system->root()->spawn(
            ActorSystem\Props::fromProducer(
                fn() => new Example\ParentActor(),
                ActorSystem\Props::withSupervisor(
                    new ActorSystem\Strategy\ExponentialBackoffStrategy(
                        new DateInterval("PT02S"),
                        new DateInterval("PT01S")
                    )
                )
            )
        );
        $system->root()->send(
            $ref,
            new Example\Message\Hello('World')
        );
    });
});
```

backoffWindowで指定した時間枠以上に再起動が行われると  
アクターの再起動時間がリセットされ、再度最初から再起動が行われます。

アクター再起動後に安定した状態になると、メッセージの受信が再開されます。  
ただしExceptionスロー時のメッセージを再処理する、という振る舞いにはならないため  
再び親アクターから再送しなければいけないことに留意してください。

## OneForOneStrategy と AllForOneStrategy

`OneForOneStrategy` と `AllForOneStrategy` は、監督者が子アクターの障害に対してどのように反応するかを指定するための戦略です。  

どちらも子アクターが完全に停止する前に失敗できる回数の制限が設定されています。

`OneForOneStrategy`は、失敗した子アクターに対して個別の戦略を適用され、  
`AllForOneStrategy`は同じ階層に位置するすべての子アクターにも適用されます。

デフォルトはOneForOneStrategyとなっていますが、  
利用シーンや負荷分散などの戦略に合わせて使い分けると良いでしょう。

OneForOneStrategyを利用する場合は`Phluxor\ActorSystem\Strategy\OneForOneStrategy`を利用しますが、  
ExponentialBackoffStrategyと異なり、再起動以外の指示が選択できるようになっています。

この選択肢・指示（ディレクティブ）として次のような違いがあります。

### Phluxor\ActorSystem\Resume

エラーが発生したアクターを再開し、メッセージの処理を続行するよう指示。

エラーは無視され、アクターはそのままの状態でメッセージ処理を続けます。

### Phluxor\ActorSystem\Restart

エラーが発生したアクターを破棄し、新しいインスタンスに置き換える（再起動）指示。

アクターの状態は初期化され、再起動後に新しいインスタンスとして動作を続けます。

### Phluxor\ActorSystem\Stop

エラーが発生したアクターを停止するよう指示。

停止されたアクターはメッセージの処理を終了しシステムから削除されます。

### Phluxor\ActorSystem\Escalate

エラーの処理をアクターの親アクターにエスカレーションするよう指示。

このディレクティブにより、エラーの処理が親に引き継がれます。

これらのディレクティブを利用してアクターの障害に対する戦略を設定するには、  
Closureを利用するか、下記の`Phluxor\ActorSystem\Supervision\DeciderFunctionInterface`を実装する必要があります。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Supervision;

use Phluxor\ActorSystem\Directive;

interface DeciderFunctionInterface
{
    /**
     * @param mixed $reason
     * @return Directive
     */
    public function __invoke(mixed $reason): Directive;
}
```

`$reason` には失敗したアクターが送信したメッセージ、ほとんどの場合はExceptionが渡されます。  

```php
Phluxor\ActorSystem\Props::withSupervisor(
    new Phluxor\ActorSystem\Strategy\OneForOneStrategy(
        15,
        new DateInterval('PT1S'),
        function (mixed $reason): ActorSystem\Directive {
            return Phluxor\ActorSystem\Directive::Restart;
        }
    )
)                
```

この例では、1秒間に15回の再起動を行うように設定する例です。  
`$reason` には例外が渡されるため、例外やメッセージ内容に合わせて適切なディレクティブを変更することも可能です。

AllForOneStrategyを利用する場合は`Phluxor\ActorSystem\Strategy\AllForOneStrategy`を利用してください。
