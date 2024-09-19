# Dead Letter

Messages that cannot be delivered to an actor become messages called "dead letters."

These dead letters are distributed via the Event Stream within the actor system and are output to the log as dead letters,  
but this is on a best-effort basis.

Even within local processes, delivery can fail (e.g., messages that couldn't reach the actor before it terminated).

Also, when using remote or clustered deployments, messages sent over unreliable networks may be lost without appearing as dead letters.

## What are the use cases?

Dead letters are primarily used for debugging purposes.

In particular, if messages cannot be sent to or received by an actor, you can check whether you are sending them incorrectly.

If an excessive number of dead letters occur, you should verify whether you have a proper understanding of [the actor's lifecycle](/en/what/lifecycle.html), whether your implementation of stopping actors is correct, or if you're continually sending unnecessary messages to non-existent actors.

## How do I Receive Dead Letters?

You can access the event stream from the actor's context and receive messages by subscribing to `Phluxor\ActorSystem\DeadLetterEvent`.

An actor that subscribes will receive all dead letters dispatched within the actor system from that point onward.

Since dead letters are not delivered over the network, please create an actor that subscribes to dead letters on each node, so they can be aggregated per node.

### Note

That it is not recommended to implement processing that subscribes to and relies on messages flowing into the dead letter queue as a trusted source of information.
