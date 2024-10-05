# Future

Futureパターンは、非同期処理を扱うときに使用される設計パターンです。

このパターンは時間のかかる操作、たとえばファイルの読み取りと書き込み、  
ネットワーク通信、データベース クエリなどを非同期で実行し、あとで結果を受け取るためのメカニズムを提供します。

Futureパターンは、将来の結果を表すオブジェクトとして機能し、  
プログラムの他の部分が結果を待たずに続行できるようにします。

Phluxorでは、Futureを使用して次の機能を提供します。

## Simplification of Asynchronous Processing

長時間実行される操作は非同期で実行されるため、  
プログラムの他の部分はブロックされることなく機能し続けることができます。

## Centralized Error Handling

非同期操作中に発生するエラーは、Futureオブジェクトを通じて1か所で処理できます。

とくにアクターモデルを導入する場合、  
複雑な非同期操作や複数の非同期タスクの処理が必要になります。

このような場合に、Futureは利便性と可読性の向上に貢献します。

## FutureとPromiseの違い

### Future?

#### Future / Definition

将来のある時点で利用可能になる値または結果を表すオブジェクト。

Future自体は値を保持しませんが、利用可能になったときに値を受け取る手段を提供します。

#### Future / Role

主に結果の「受信者」として機能します。

Futureは結果がいつ利用可能になるかはわかりませんが、結果の準備ができたときに通知する方法を提供します。

### Promise?

#### Promise / Definition

結果をFutureに「提供」する役割を持つオブジェクト。

Promiseは結果を解決または拒否する役割を持ちます。

#### Promise / Role

主に結果の「プロデューサー」として機能します。

Promiseは、非同期操作の完了時に結果をFutureに渡します。

Phluxorでは結果の受信者として、結果がいつ準備できるかを気にすることなく、  
結果が利用可能になったときに通知するメカニズムを提供します。

## Usage

Futureの使用方法は、requestとsendに似ています。

PhluxorでFutureを利用する場合は `requestFuture` を使用してメッセージを送信するだけです。

この時点で受信側のアクターが送信者に返信できるように、Futureの`Ref`が組み込まれます。

Futureオブジェクトは `wait()` または `result()` を使用して、  
応答が到着するか実行がタイムアウトするまで待機します。

```php
$system = ActorSystem::create();
$pid = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
$future = $system->root()->requestFuture($pid, new YourRequest(), 1);
var_dump($future->result());        
```

もちろん、子アクターでも同様に `requestFuture` メソッドを利用できます。

```php
public function receive(ContextInterface $context): void
{
    $msg = $context->message();
    switch (true) {
        case $msg instanceof StartsClass:
            $ref = $context->spawn(
                Props::fromProducer(
                    fn() => new TeacherActor(
                        $this->students, $context->self()
                    )
                )
            );
            $future = $context->requestFuture(
                $ref, 
                new PrepareTest(['subject' => $msg->getSubject()]),
                1
            );
            var_dump($future->result());
            break;
    }
}
```

3番目の引数はタイムアウト期間を秒単位で指定します。

### pipeTo

`PipeTo`を使用すると、  
Futureがタイムアウトする前に応答が返された場合にFutureは複数の宛先に応答を送信できます。

これによりアクターがブロックされることはないため、着信メッセージは到着時に処理されます。

実行がタイムアウトすると応答はデッドレターに送信され、呼び出し元のアクターには通知されません。

```php
$system = ActorSystem::create();
$pid = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
$future = $system->root()->requestFuture($pid, new YourRequest(), 1);
$future->pipeTo($otherActorOneRef);
$future->pipeTo($otherActorTwoRef);
$future->pipeTo($otherActorThreeRef);
```

子アクターでも同様に `pipeTo` メソッドを利用できます。

```php
class YourActor implements ActorInterface
{
    public function __construct(
        private Ref $otherActorOneRef,
    ) {}

    public function receive(ContextInterface $context): void
    {
        $msg = $context->message();
        switch (true) {
            case $msg instanceof StartsClass:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new TeacherActor(
                            $this->students, $context->self()
                        )
                    )
                );
                $future = $context->requestFuture(
                    $ref, 
                    new PrepareTest(['subject' => $msg->getSubject()]),
                    1
                );
                $future->pipeTo($this->otherActorOneRef);
                var_dump($future->result());
                break;
        }
    }
}
```
