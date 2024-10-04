# Spawning Actors

アクターは任意の数のアクターを生成できます。

生成されたアクターは子アクターとなり、それぞれがさらに自分の子アクターを生成できます。  
（この場合、生成したアクターが親アクターとなります）

これにより、アクターは階層構造を持つことができます。

`ActorSystem`はこの階層のホストとして機能し、その直下に生成されるルートアクターはトップレベルのアクターとなります。

子アクターのライフサイクルは親アクターのライフサイクルに依存します。

子アクターはいつでも停止や再起動が可能ですが、  
親アクターが停止するとその子アクターも停止するため、子アクターが親アクターを超えて存続することはできません。

## Spawn

アクターを生成する際、ルートアクターは`root()`から作成され、子アクターは`context`から作成されます。

子アクターやルートアクターを作成するには、`spawn`メソッドを使用します。

最初のパラメーターは`Phluxor\ActorSystem\Props`オブジェクトです。
自動生成された名前でアクターを生成します。

*[Props](/ja/features/props.html)はアクターを作成するための構成オブジェクトです。

### root

```php
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

$system = ActorSystem::create();

$spawned = $system->root()->spawn(
    Props::fromProducer(
        fn() => new YourActor()
    )
);
```

### child

```php
namespace App\ActorSystem;

use App\Command\CreateUser;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        if ($msg instanceof CreateUser) {
            $context->spawn(
                Props::fromProducer(
                    fn() => new YourChildActor()
                )
            );
        }
    }
}

```

ルートアクターと子アクターを生成する方法は同じです。

## SpawnNamed

`spawnNamed`メソッドは、名前付きアクターを作成するために使用されます。

2番目のパラメーターにはユニークな名前が必要です。

事前に名前を付けてアクターを作成することで、

意図したアクターにメッセージを送信したり、状態をより簡単に管理できます。

もしその名前がすでに使用されている場合は、

例外（`Phluxor\ActorSystem\Exception\NameExistsException`）がスローされます。

```php
namespace App\ActorSystem;

use App\Command\CreateUser;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        if ($msg instanceof CreateUser) {
            $context->spawnNamed(
                Props::fromProducer(
                    fn() => new YourChildActor()
                ),
                'unique-name'
            );
        }
    }
}

```

## SpawnPrefix

`spawnPrefix`メソッドは、プレフィックス付きのアクターを作成するために使用されます。

プレフィックスはアクターの名前に追加されます。

```php
namespace App\ActorSystem;

use App\Command\CreateUser;
use Phluxor\ActorSystem\Context\ContextInterface;
use Phluxor\ActorSystem\Message\ActorInterface;
use Phluxor\ActorSystem\Props;

class YourActor implements ActorInterface
{
    public function receive(ContextInterface $context): void
    {
        if ($msg instanceof CreateUser) {
            $context->spawnPrefix(
                Props::fromProducer(
                    fn() => new YourChildActor()
                ),
                'prefix-'
            );
        }
    }
}

```
