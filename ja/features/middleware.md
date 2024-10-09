# Middleware

ミドルウェアは、アクターのメッセージを処理する前後にフックを追加するための機能です。  

受信メッセージと送信メッセージの両方にミドルウェアを追加できます。  
PhluxorではReceiverMiddlewareとSenderMiddlewareと呼ばれ、  

`Phluxor\ActorSystem\Props`を利用してミドルウェアの追加が可能です。

## ReceiverMiddleware

受信メッセージのミドルウェアは、アクターがメッセージを受信する前後にフックを追加します。  

`Props::withReceiverMiddleware`メソッドを使用して、  
アクターの受信メッセージにミドルウェアを追加できます。  
任意の数だけ追加でき、追加された順に実行されます。  

`Props::withReceiverMiddleware`には、  
`Phluxor\ActorSystem\Props\ReceiverMiddlewareInterface`を実装するクラスを渡す必要があり、  
そのクラスは`Phluxor\ActorSystem\Message\ReceiverFunctionInterface`を返す`__invoke`メソッドを実装する必要があります。  

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Props;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;

interface ReceiverMiddlewareInterface
{
    /**
     * @param Closure(ReceiverInterface|ContextInterface, MessageEnvelope): void|ReceiverFunctionInterface $next
     * @return ReceiverFunctionInterface
     */
    public function __invoke(
        Closure|ReceiverFunctionInterface $next
    ): ReceiverFunctionInterface;
}
```

ReceiverFunctionInterfaceは、アクターがメッセージを受信する際に実行される関数です。  
次のミドルウェアをよびだすために、`$next`引数を使用します。  

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;

interface ReceiverFunctionInterface
{
    /**
     * @param ReceiverInterface|ContextInterface $context
     * @param MessageEnvelope $messageEnvelope
     * @return void
     */
    public function __invoke(
        ReceiverInterface|ContextInterface $context,
        MessageEnvelope $messageEnvelope
    ): void;
}
```

`ReceiverFunctionInterface`は、アクターがメッセージを受信する際に実行される関数です。  
`$context`はアクターのコンテキストを表し、`$messageEnvelope`は受信したメッセージを表します。

これによりアクターが受信したメッセージに対しての処理をカスタマイズできます。  

実装例は以下の通りです。  

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\ReceiverInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;

readonly class ReceiverFactory implements ReceiverFunctionInterface
{
    public function __construct(
        private Closure|ReceiverFunctionInterface $next
    ) {
    }

    public function __invoke(
        ContextInterface|ReceiverInterface $context,
        MessageEnvelope $messageEnvelope
    ): void {
        var_dump('before receiver');
        $next = $this->next;
        $next($context, $messageEnvelope); // next middleware or actor receive
        var_dump('after receiver');
    }
}

```

上記の例では、アクターがメッセージを受信する前後にログを出力します。  

ミドルウェアを追加するためのクラスは以下のようになります。

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Message\ReceiverFunctionInterface;
use Phluxor\ActorSystem\Props\ReceiverMiddlewareInterface;

class ReceiverMiddleware implements ReceiverMiddlewareInterface
{
    public function __invoke(ReceiverFunctionInterface|Closure $next): ReceiverFunctionInterface
    {
        return new ReceiverFactory($next);
    }
}

```

これらをアクターに適用するためには、`Props::withReceiverMiddleware`メソッドを使用します。  

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Example\Middleware\ReceiverMiddleware;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new ChildActor(),
                        Props::withReceiverMiddleware(
                            new ReceiverMiddleware()
                        )
                    )
                );
                $context->send($ref, $message);
                break;
        }
    }
}

```

複数のミドルウェアを追加すると下記のように実行されます。  

```bash
string(15) "before receiver" // first receiver middleware
string(16) "before receiver2" // second receiver middleware
string(15) "after receiver2" // second receiver middleware
string(14) "after receiver" // first receiver middleware
```

## SenderMiddleware

送信メッセージのミドルウェアは、アクターがメッセージを送信する前後にフックを追加します。

`Props::withSenderMiddleware`メソッドを使用して、  
アクターの送信メッセージにミドルウェアを追加できます。  
`ReceiverMiddleware`と同様に任意の数だけ追加でき、追加された順に実行されます。

`Props::withSenderMiddleware`には、  
`Phluxor\ActorSystem\Props\SenderMiddlewareInterface`を実装するクラスを渡す必要があり、  
そのクラスは`Phluxor\ActorSystem\Message\SenderFunctionInterface`を返す`__invoke`メソッドを実装する必要があります。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Props;

use Closure;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Ref;

interface SenderMiddlewareInterface
{
    /**
     * @param Closure(SenderInterface|ContextInterface, Ref, MessageEnvelope): void|SenderFunctionInterface $next
     * @return SenderFunctionInterface
     */
    public function __invoke(Closure|SenderFunctionInterface $next): SenderFunctionInterface;
}
```

`SenderFunctionInterface`は、アクターがメッセージを送信する際に実行される関数です。  
次のミドルウェアをよびだすために、`$next`引数を使用します。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Ref;

interface SenderFunctionInterface
{
    /**
     * @param SenderInterface $context
     * @param Ref|null $target
     * @param MessageEnvelope $messageEnvelope
     * @return void
     */
    public function __invoke(
        SenderInterface $context,
        Ref|null $target,
        MessageEnvelope $messageEnvelope
    ): void;
}

```

`SenderFunctionInterface`は、アクターがメッセージを送信する際に実行される関数です。  
`$context`はアクターのコンテキストを表し、`$target`は送信先のアクターを表します。  
`$messageEnvelope`は送信するメッセージを表します。

これによりアクターが送信するメッセージに対しての処理をカスタマイズできます。  
基本的な実装方法は受信メッセージのミドルウェアと同様です。  

以下に例を示します。

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Context\SenderInterface;
use Phluxor\ActorSystem\Message\MessageEnvelope;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Ref;

readonly class SenderFactory implements SenderFunctionInterface
{
    public function __construct(
        private SenderFunctionInterface|Closure $next
    ) {
    }

    public function __invoke(SenderInterface $context, ?Ref $target, MessageEnvelope $messageEnvelope): void
    {
        var_dump('before sender');
        $next = $this->next;
        $next($context, $target, $messageEnvelope);
        var_dump('after sender');
    }
}

```

ミドルウェアを追加するためのクラスは以下のようになります。

```php
<?php

declare(strict_types=1);

namespace Example\Middleware;

use Closure;
use Phluxor\ActorSystem\Message\SenderFunctionInterface;
use Phluxor\ActorSystem\Props\SenderMiddlewareInterface;

class SenderMiddleware implements SenderMiddlewareInterface
{
    public function __invoke(SenderFunctionInterface|Closure $next): SenderFunctionInterface
    {
        return new SenderFactory($next);
    }
}

```

これらをアクターに適用するためには、`Props::withSenderMiddleware`メソッドを使用します。  

```php
<?php

use Phluxor\ActorSystem;

$system = ActorSystem::create();
$ref = $system->root()->spawn(
    ActorSystem\Props::fromProducer(
        fn() => new Example\ParentActor(),
        ActorSystem\Props::withSenderMiddleware(
            new Example\Middleware\SenderMiddleware()
        )
    )
);
```

アクターコンテキスト内での利用方法も同様です。  
もちろん、複数のミドルウェアを追加することも可能です。  

### Middlewareの利用例

ミドルウェアは、アクターのメッセージを処理する前後にフックを追加するための機能ですので、  
Event SourcingやCQRSの実装にも利用できます。

たとえば、アクターがメッセージを受信する前後にイベントを発行するミドルウェアとしてメッセージの永続化なども実装できます。  

これらの仕組みを詳しく知りたい場合は、永続化やイベントソーシングの章を参照してください（ドキュメント準備中）。

## MailboxMiddleware

MailboxMiddlewareは、アクターのメッセージがMailboxに到達した際にフックを追加するための機能です。  

MailboxMiddlewareも他ミドルウェア同様に`Phluxor\ActorSystem\Props`を利用して追加できます。  

MailboxMiddlewareを利用するには`Phluxor\ActorSystem\Mailbox\MailboxMiddlewareInterface`を実装するクラスを渡す必要があり、  
`Props::withMailboxProducer`メソッドを使用してMailboxMiddlewareを追加します。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Mailbox;

interface MailboxMiddlewareInterface
{
    public function mailboxStared(): void;

    public function messagePosted(mixed $message): void;

    public function messageReceived(mixed $message): void;

    public function mailboxEmpty(): void;
}

```

`mailboxStarted`はMailbox自体が起動した時にコールされるメソッドです。

`messagePosted`はMailboxに対してシステムメッセージ・ユーザーメッセージが送信されるとコールされます。  
コールされたあとにMailboxにpushされます。

`messageReceived`はMailboxにシステムメッセージ・ユーザーメッセージが到達・受け取るとコールされます。  

`mailboxEmpty`はMailboxが空になるたびにコールされます。

MailboxのカスタマイズはPhluxorの基本的な知識が必要ですので、  
アクターモデルやPhluxorについて十分な知識がない限り、カスタマイズはオススメしませんが、  
ミドルウェアの追加は簡単に行えます。

PhluxorのMailboxはデフォルトではUnbounded Mailboxを使用しています。  
このMailboxにミドルウェアを追加するには以下のようにします。

```php
<?php

declare(strict_types=1);

namespace Example;

use Example\Message\Hello;
use Example\Middleware\MailboxMiddleware;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Mailbox\Unbounded;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class ParentActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        $message = $context->message();
        switch (true) {
            case $message instanceof Hello:
                $ref = $context->spawn(
                    Props::fromProducer(
                        fn() => new ChildActor(),
                        Props::withMailboxProducer(
                            new Unbounded(new MailboxMiddleware())
                        )
                    )
                );
                $context->send($ref, $message);
                break;
        }
    }
}

```
