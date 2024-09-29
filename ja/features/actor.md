# Actors

アクターはアクターモデルの基本的な単位です。

アクターは状態と動作、メッセージを受信するためのメールボックス、  
子アクター、エラーを処理するためのスーパーバイザー戦略をカプセル化するオブジェクトです。

すべてのアクターは、
アクターリファレンス（PhluxorではRefオブジェクト）の背後にユニークのアドレスを持ち、  
Refのみを介してメッセージを送信し、他のアクターと通信します。

`Phluxor\ActorSystem` クラスは、アクターを作成するための始まりです。

*Phluxorではswoole extensionが必須となります*

アクターシステムの起動の仕方は以下の通りです。

```php
$system = \Phluxor\ActorSystem::create();
```

アクターシステムは、アクターのライフサイクルを管理する中心に位置するものです。

## アクターの実装方法

アクターは、`Phluxor\ActorSystem\Message\ActorInterface` インターフェイスを実装するオブジェクトです。

すべてのアクターにはコンテキストを受け取る `receive` メソッドのみがあります。

```php
<?php

declare(strict_types=1);

namespace Phluxor\ActorSystem\Message;

use Phluxor\ActorSystem\Context\ContextInterface;

interface ActorInterface
{
    /**
     * Receives a context.
     *
     * @param ContextInterface $context The context to receive.
     * @return void
     */
    public function receive(ContextInterface $context): void;
}

```

`ContextInterface`インターフェイスはアクターのメールボックスへのアクセスを提供します。

コンテキストは、この他にもアクターシステムへのアクセスや、  
自アクターのアドレス、親アクター情報や子アクター情報、スーパーバイザー戦略などの情報を提供します。  

なおアクターはreturnを持たず、`void`を返すことに注意してください。

すべてのアクターは非同期で、並行で動作するため、戻り値を持つことはできません。

アクターはメッセージパッシングを通じて非同期で他のアクターと通信するため、アクターを直接呼び出すことや、操作することはできません。
