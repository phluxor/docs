# Timeout for receiving messages

`setReceiveTimeout` メソッドを呼び出すことで、メッセージに応答する前の処理にタイムアウトを設定できます。

タイムアウトに達すると、アクターは `Phluxor\ActorSystem\Message\ReceiveTimeout` メッセージを受信します。

このメッセージを使用してタイムアウトに対する応答をトリガーできます。

```php
public function receive(ContextInterface $context): void
{
    $message = $context->message();
    switch (true) {
        case $message === 'hello':
            $context->setReceiveTimeout(new DateInterval('PT1S'));
            // any process that takes more than 1 second will be terminated
            break;
        case $message instanceof ActorSystem\Message\ReceiveTimeout:
            // receive timeout message
            break;
    }
}
```

## Note

このタイムアウトは、メッセージがメールボックスに到達した直後に  
`ReceiveTimeout` メッセージをメールボックスに挿入する可能性があります。

一度設定すると、受信タイムアウトは有効なままです。

つまり、非アクティブな期間の後も繰り返し発生し続けることになります。

タイムアウトをオフにするには、`setReceiveTimeout` に `new \DateInterval('PT0S')` を渡します。

```php
public function receive(ContextInterface $context): void
{
    $message = $context->message();
    switch (true) {
        case $message === 'hello':
            $context->setReceiveTimeout(new DateInterval('PT1S'));
            // any process that takes more than 1 second will be terminated
            break;
        case $message === 'reset':
            // reset the timeout
            $context->setReceiveTimeout(new DateInterval('PT0S'));
            break;
        case $message instanceof ActorSystem\Message\ReceiveTimeout:
            // no timeout
            break;
    }
}
```

## NoInfluence

受信タイムアウトタイマーに影響を与えないメッセージに対して、  
タイムアウトをリセットせずにアクターにメッセージを送信する方法があります。

これを行うには、  
メッセージに `Phluxor\ActorSystem\Message\NotInfluenceReceiveTimeoutInterface` を実装します。

例：

```php
<?php

declare(strict_types=1);

namespace Acme\Message;

use Phluxor\ActorSystem\Message\NotInfluenceReceiveTimeoutInterface;

class NoInfluence implements NotInfluenceReceiveTimeoutInterface
{
}
```
