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

When an actor fails to process a message, i.e. returns an error from it’s receiver,  
the actor’s mailbox will be suspended in order to halt further processing,  
and the failure will be escalated to whoever is supervising the actor (see supervision).  
Depending on the directive the supervisor decides to apply, the actor may be either resumed, restarted or stopped.

