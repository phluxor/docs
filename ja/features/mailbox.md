# Mailbox

アクターはメッセージを通じてのみ相互作用します。

これらのメッセージはアクターに直接送られるのではなく、各アクターが持つメールボックスに送信されます。  
アクターがメッセージを処理する準備が整うと、  
メールボックスからメッセージがアクターにプッシュされます。

デフォルトのメールボックスは、システムメッセージとユーザーメッセージの2つのメッセージキューで構成されています。  
システムメッセージはシステム内部で使用され、たとえばクラッシュ時のメールボックスの一時停止や再開、  
また各アクターの管理（開始、停止、再起動など）にも使用されます。  

ユーザーメッセージは開発者が自由に使用できるメッセージです。  
これらのメッセージはアクターコンテキストからアクセスできます。

メールボックス内のメッセージは一般的にFIFO（先入れ先出し）順で配信されますが、  
システムメッセージが優先され、ユーザーメッセージの前に処理されます。

アクターのメールボックスには次のルールが適用されます:

- メッセージ送信は複数のプロデューサーによって同時に行うことができます（MPSC: マルチプロデューサー、シングルコンシューマー）。

- メッセージ受信は各アクターに対して順番に行われます。システムメッセージのような特別なメールボックスには異なるルールを適用することも可能です。

- メールボックスはアクター間で共有できません。内部的にも共有されません。

デフォルトのメールボックスは無制限なので、  
スループットなどの調整のためにメールボックスのキューに任意の制限を指定できます。

## Changing the Mailbox

特定のメールボックス実装を使用するには、`Phluxor\ActorSystem\Props` を使ってカスタマイズできます。  
その場合は`Phluxor\ActorSystem\Props::withMailboxProducer` メソッドを使用して指定します。

```php
Props::fromProducer(fn() => new YourActor(),
    Props::withMailboxProducer(YourMailboxProducer)
);
```

`Phluxor\ActorSystem\Mailbox\MailboxProducerInterface` インターフェイスを実装することもできます。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

interface MailboxProducerInterface
{
    /**
     * @return MailboxInterface
     */
    public function __invoke(): MailboxInterface;
}
```

独自のMailboxを利用したい場合は `Phluxor\ActorSystem\Mailbox\MailboxInterface`インターフェイスを実装します。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

use Phluxor\ActorSystem\Dispatcher\DispatcherInterface;

interface MailboxInterface
{
    /**
     * @param mixed $message
     * @return void
     */
    public function postUserMessage(mixed $message): void;

    /**
     * @param mixed $message
     * @return void
     */
    public function postSystemMessage(mixed $message): void;

    /**
     * @return void
     */
    public function start(): void;

    /**
     * @return int
     */
    public function userMessageCount(): int;

    /**
     * @param MessageInvokerInterface $invoker
     * @param DispatcherInterface $dispatcher
     * @return void
     */
    public function registerHandlers(
        MessageInvokerInterface $invoker,
        DispatcherInterface $dispatcher
    ): void;
}
```

## Unbounded Mailbox

無制限のメールボックスは、便利なデフォルトオプションとして利用できます。  

しかし、アクターがメッセージを処理する速度よりも速く大量のメッセージがメールボックスに追加されると、  
メモリ不足や不安定な動作を引き起こす可能性があります。  
そのため、制限付きのメールボックスを指定できます。  

制限付きメールボックスを使用する場合、  
メールボックスがいっぱいになると新しいメッセージは[DeadLetter](/ja/features/deadletter.html)キューに転送されます。

## Mailbox Instrumentation

### Dispatchers and Invokers

メールボックスには、DispatcherとInvokerの2つのコンポーネントを設定できます。  
アクターが作成されると、Invokerがアクターコンテキストとなり、  
Dispatcherは `Phluxor\ActorSystem\Props` から生成されます。

### Mailbox Invoker

メールボックスがキューからメッセージを取得すると、  
そのメッセージを登録されたInvokerに渡して処理します。  
アクターの場合、アクターコンテキストがメッセージにアクセスし、アクターの`receive`メソッドを呼び出して処理を行います。

メッセージ処理中にエラーが発生した場合、  
メールボックスはエラーを登録されたInvokerにエスカレートし、  
Invokerはアクターの再起動などの処理を実行しながら処理を続行します。

### Mailbox Dispatchers

メッセージがメールボックスに配信されると、Dispatcherがメールボックスキュー内のメッセージ処理をスケジュールします。

デフォルトのDispatcherは、  
Swooleのコルーチンを使用する `Phluxor\ActorSystem\Dispatcher\CoroutineDispatcher` を介してメッセージを処理します。

さらに、コルーチンを使用しない `Phluxor\ActorSystem\Dispatcher\SynchronizedDispatcher` もあります。  

たとえば、アクターが高負荷で大量のリソースを消費する場合などは同期ディスパッチャーを使用できます。

変更する場合はアクターの作成時に `Phluxor\ActorSystem\Props` を使用してディスパッチャーを指定します。

変更方法は以下の通りです:

```php
use Phluxor\ActorSystem\Dispatcher\SynchronizedDispatcher;
use Phluxor\ActorSystem\Props;

Props::fromProducer(fn() => new YourActor(),
    Props::withDispatcher(new SynchronizedDispatcher(300))
);
```

## Note

開発者は独自のカスタムディスパッチャーを実装して使用できますが、  
これは特別なケースに限り検討してください。

ディスパッチャーを正しく使用するためには、  
Phluxorのメールボックスを含むアクター関連の動作について一定の理解が不可欠です。

カスタムディスパッチャーは、必ずしもパフォーマンス問題を解決するための有効な手段とは限りません。
