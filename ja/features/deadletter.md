# Dead Letter

アクターに配信できなかったメッセージは「デッドレター」と呼ばれるメッセージになります。

これらのデッドレターは、アクターシステム内のイベントストリームを介して配信され、  
デッドレターとしてログに出力されますが、これはベストエフォートベースです。

ローカルプロセス内でも、配信が失敗する場合があります（例: アクターが終了する前に届かなかったメッセージ）。

また、リモートやクラスター構成を使用する場合、  
信頼性の低いネットワーク上で送信されたメッセージは、デッドレターとして現れることなく失われる可能性があります。

## ユースケースは何ですか？

デッドレターは主にデバッグ目的で使用されます。

とくに、アクターにメッセージが送信できない、または受信されない場合、送信が正しく行われているか確認するために役立ちます。

デッドレターが過剰に発生する場合は、[アクターのライフサイクル](/ja/what/lifecycle.html) を適切に理解しているか、  
アクターの停止に関する実装が正しいか、  
または存在しないアクターに対して不要なメッセージを継続的に送信していないか確認する必要があります。

## デッドレターを受信するにはどうすればよいですか？

アクターのコンテキストからイベントストリームにアクセスし、  
`Phluxor\ActorSystem\DeadLetterEvent` をサブスクライブすることでメッセージを受信できます。

サブスクライブしたアクターは、その時点からアクターシステム内で配信されるすべてのデッドレターを受信します。

デッドレターはネットワークを介して配信されないため、  
各ノードでデッドレターをサブスクライブするアクターを作成し、ノードごとに集約できるようにしてください。

### Note

デッドレターキューに流れ込むメッセージにサブスクライブし、  
それを信頼できる情報源として依存する処理を実装することは推奨されません。

## Usage

```php
use Phluxor\ActorSystem;

$system = ActorSystem::create();
$system->getEventStream()?->subscribe(function (mixed $event): void {
    if ($event instanceof ActorSystem\DeadLetterEvent) {
        // do something with dead letter
    }
});
```

子アクターでもデッドレターにサブスクライブできます。

```php
use Phluxor\ActorSystem;

// 

public function receive(ContextInterface $context): void
{
    $context->getEventStream()?->subscribe(function (mixed $event): void {
        if ($event instanceof ActorSystem\DeadLetterEvent) {
            // do something with dead letter
        }
    });
}
```
