# Actor Lifecycle

The actor lifecycle in Phluxor represents the sequence of events from the creation to the termination of an actor. 

In a basic scenario, an actor is first instantiated,  
a Started message is sent,  
and the actor remains alive until the application shuts down.

## Stopping an Actor

You can stop an actor using various methods:

Sending a Stop message from the context.  

Calling the stop method on a `Phluxor\ActorSystem\ActorContext` instance.  

## Failure and supervision

If an actor fails to process a message,  
the actor's mailbox is temporarily paused to stop processing.

This failure is escalated to other actors or the parent actor supervising the failed actor (see supervision).  
Depending on the instructions decided by the supervisor,  
the actor may automatically resume, restart, or stop.

### Resume

The mailbox resumes processing,  
and the actor continues execution from the message that previously failed.

### Stop

When an actor is stopped,  
the object that was functioning as the actor is discarded.  
Before the stop, the actor receives a stop message, and after being stopped,  
it also receives the stop message.

### Restart

A restart message is sent to the actor,  
notifying it that a restart is being attempted.  
After this is processed, the actor stops, and the object that was functioning as the actor is discarded and recreated.  
The mailbox is then resumed,  
and the newly created actor receives a start message.  
Past messages cannot be retrieved after a restart.

## Handling events

`Phluxor\ActorSystem\Message\Started` is the first message an actor receives after being spawned or restarted.  
If you need to set the initial state of the actor,  
such as loading data from a database, you should handle this message.

`Phluxor\ActorSystem\Message\Restarting` is sent when an actor is about to restart,  
and `Phluxor\ActorSystem\Message\Stopping` is sent when an actor is about to stop.  
In both cases, the actor's object will be discarded,  
so if you need to execute any logic to ensure a proper shutdown (e.g., saving the state to a database, sending a message to a messaging system, etc.),  
you should handle these messages.

`Phluxor\ActorSystem\Message\Stopped` is sent when an actor has stopped,  
and the actor and its associated objects have been detached from the system.  
At this stage, the actor can no longer send or receive messages,  
and after the message is processed, the object is completely discarded.
