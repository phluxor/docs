# Future

The Future Pattern is a design pattern used when dealing with asynchronous processing.

This pattern provides a mechanism to execute time-consuming operations (such as file reading and writing, network communication, database queries, etc.) asynchronously and receive the results later.

The Future Pattern functions as an object representing a future result,  
allowing other parts of the program to proceed without having to wait for the result.

In Phluxor, we use Futures to provide the following:

## Simplification of Asynchronous Processing

Long-running operations are executed asynchronously, allowing other parts of the program to continue functioning without being blocked.

## Centralized Error Handling

Errors that occur during asynchronous operations can be handled in one place through the Future object.

Especially when introducing the actor model,  
complex asynchronous operations and handling multiple asynchronous tasks become necessary.  
Futures contribute to improving convenience and readability in such cases.

## The differences between Future and Promise

### Future?

#### Future / Definition

An object that represents a value or result that will be available at some point in the future.

A Future itself does not hold the value but provides a means to receive it when it becomes available.

#### Future / Role

Primarily functions as the "receiver" of the result.

A Future does not know when the result will be available but offers a way to be notified when the result is ready.

### Promise?

#### Promise / Definition

An object that is responsible for "providing" the result to the Future.

A Promise is responsible for resolving or rejecting the result.

#### Promise / Role

Primarily functions as the "producer" of the result.

A Promise passes the result to the Future upon the completion of the asynchronous operation.

In Phluxor, as the receiver of the result,  
we provide a mechanism that notifies you when the result becomes available without the need to be concerned about when it will be ready.

## Usage

The usage of Future is similar to request and send.

When using a Future in Phluxor, you simply send a message using `requestFuture`.

At this point, the `Ref` of the Future is embedded so that the receiving actor can reply to the sender.

The Future object uses `wait()` or `result()` to wait until the response arrives or the execution times out.

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

ofcourse, child actor can also use `requestFuture` method.

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

The third argument specifies the timeout period in seconds.

### pipeTo

By using PipeTo, a Future can send responses to multiple destinations if the response is returned before the Future times out.

This does not block the actor, so incoming messages are processed as they arrive.

If the execution times out, the response is sent to the dead letter mailbox, and the calling actor is not notified.

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

or child actor can also use `pipeTo` method.

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
