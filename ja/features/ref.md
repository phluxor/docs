# Ref / Actor Referece

一般的にアクターモデルのツールキットでは、アクターを作成するときに直接参照することはできません。  
代わりに、アクターのメールボックスへの参照を取得し、  
そのメールボックスにメッセージを送信してアクターと通信します。
この参照は `Phluxor\ActorSystem\Ref`、別名 `Actor Reference`（PhluxorではRef）と呼ばれます。

この `Ref` はシリアル化可能で低コストでネットワーク経由で送信できます。  
ローカルでもリモートでも、`Ref`を介してアクターにメッセージを送信できます。

## Overview

Phluxorでは、アクターはメッセージを送信して互いに通信します。  
そのため、メッセージを送信するには別のアクターへの参照を必要とします。

![send message to actor](/images/ref/send_ref.png "send message to actor")

アクターにメッセージを送信する場合、アクターを識別することが重要です。

Phluxorでは、アクターを識別するために一意の `Ref` が使用されます。  
この`Ref`は新しいアクターが作成されると、そのアクターへの参照として `Ref` が返されます。  
Phluxorは、どのアクターにも割り当てられていない名前をアクターに自動的に割り当てます。

アクターが停止すると、その `Ref` は無効なものとなります。

![send message to actor with ref](/images/ref/send_to_actor_with_ref.png "send message to actor with ref")

## Create Actor

アクターが生成されると、`Ref`が作成されて返却されます。

### root

```php
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\Props;

$system = ActorSystem::create();

// spawn a root actor
// return a Ref
$ref = $system->root()->spawn(
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
            // spawn a child actor
            // return a Ref
            $ref = $context->spawn(
                Props::fromProducer(
                    fn() => new YourChildActor()
                )
            );
        }
    }
}

```

## Create Manually

`Phluxor\ActorSystem\Ref` は手動で作成することもできます。

これはアクターの識別子として機能することを意図しているため、  
名前が適切である限り、手動で作成して使用しても問題はありません。

Refは内部で `Phluxor\ActorSystem\ProtoBuf\Pid` オブジェクトを使用して作成されます。

手動で作成する場合はいくつか決まりがありますが、  
ローカルでのみ構成されるアクターの場合は `address` を `ActorSystem::LOCAL_ADDRESS` に設定してください。  

このアドレスは、アクターがローカルかリモートかを判断するために使用されるためです。  
アドレスが `ActorSystem::LOCAL_ADDRESS` でない場合、  
内部的に対象のアクターはリモートであると見なされます。

```php
use Phluxor\ActorSystem;
use Phluxor\ActorSystem\ProtoBuf\Pid;
use Phluxor\ActorSystem\Ref;

new Ref(new Pid([
    'id' => 'special-id',
    'address' => ActorSystem::LOCAL_ADDRESS,
]));
```

## Send Message

`send`はアクターにメッセージを送信するための、ブロッキングを行わない、Fire and Forget方式です。

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
                // send a message to the actor
                $context->send($ref, new PrepareTest(['subject' => $msg->getSubject()]));
                break;
        }
    }
```

## Request Message

`request` は `send` と非常に似ていますが、受信側のアクターが送信者に返信できるように送信側の `Ref` が含まれています。

2つのアクター間でリクエスト/返信の通信が必要な場合にのみ使用してください。

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
                // request a message to the actor
                $context->request($ref, new PrepareTest(['subject' => $msg->getSubject()]));
                break;
        }
    }
```
