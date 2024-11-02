# RoundRobin

`RoundRobin`は、メッセージを順番にアクターにルーティングするルーターです。

![Round Robin](/images/router/round_robin.png "round robin")

`RoundRobin`は、`RoundRobinPool`と`RoundRobinGroup`の2つのタイプがあります。  
[プールとグループ](/ja/features/router.html)

`RoundRobinPool`と`RoundRobinGroup`は、  
メッセージをラウンドロビン順にルーティングするルーターです。  
これは、複数のワーカーアクターにメッセージを配信するルーターとして一番簡単な方法です。

## 使い方

ラウンドロビンルーターを使用して5つのワーカーを展開する例を示します。

```php
use Phluxor\ActorSystem;
use Phluxor\Router\RoundRobin\PoolRouter;

run(function () {
    go(function () {
        $system = ActorSystem::create();
        $props = PoolRouter::create(
            5,
            ActorSystem\Props::withProducer(
                fn() => myActor()
            )
        );
        $router = $system->root()->spawn($props);
        $future = $system->root()->requestFuture($router, new Hello(), 1000);
        $v = $future->result()->value();
        var_dump($v);
    });
});
```
