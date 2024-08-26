# Built-In Messages

Here is a list of messages used internally in Phluxor.

## Supervision

### Phluxor\ActorSystem\Message\Failure

This message is sent when an actor throws an exception.  
The actor receives this message to handle the error.  
At this time, the mailbox of the respective actor is suspended,  
and message processing is temporarily halted.

### Phluxor\ActorSystem\Message\ResumeMailbox

This message is sent when the mailbox of an actor is in a suspended state.  
The actor receives this message to resume the processing of the mailbox.  
It is also sent internally when the actor restarts.

### Phluxor\ActorSystem\Message\SuspendMailbox

This message is sent to an actor to temporarily halt the processing of its mailbox.  
It is sent internally when the actor receives a `Phluxor\ActorSystem\Message\Failure` message,  
placing the mailbox in a suspended state.

### Phluxor\ActorSystem\ProtoBuf\Watch

This message is used by an actor to monitor another actor.  
It is sent internally when using futures.  
The actor receiving this message will monitor the state of the target actor  
and will receive a `Phluxor\ActorSystem\ProtoBuf\Terminated` message when the target actor terminates.  
It is used by actors responsible for monitoring,  
such as routers and supervisors, but it can also be used optionally.

### Phluxor\ActorSystem\ProtoBuf\Unwatch

This message is sent to unwatch another actor.  
It is sent internally by the actor that received the `Phluxor\ActorSystem\ProtoBuf\Watch` message when it stops monitoring the target actor.

## Lifecycle

### Phluxor\ActorSystem\ProtoBuf\PoisonPill

This message is used to terminate an actor and stop its message queue.  
The actor is terminated after the messages in its mailbox are processed. It is sent as a normal message,  
not as a system message, and is processed in the order received.

### Phluxor\ActorSystem\Message\Restart

This message is used to restart an actor.

### Phluxor\ActorSystem\Message\Restarting

This message is sent during the restarting of an actor.

### Phluxor\ActorSystem\Message\Started

This message is sent when an actor is started.  
It is sent internally when an actor is spawned.

### Phluxor\ActorSystem\ProtoBuf\Stop

This message is used to immediately stop an actor,  
regardless of whether there are user messages in the mailbox.

### Phluxor\ActorSystem\Message\Stopping

This message indicates that an actor is stopping.

### Phluxor\ActorSystem\Message\Stopped

This message is sent when an actor is stopped.

### Phluxor\ActorSystem\ProtoBuf\Terminated

This message is sent when a watched actor terminates.
