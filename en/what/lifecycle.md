# Actor Lifecycle

The actor lifecycle in Phluxor represents the sequence of events from the creation to the termination of an actor. 

In a basic scenario, an actor is first instantiated,  
a Started message is sent,  
and the actor remains alive until the application shuts down.

## Stopping an Actor

You can stop an actor using various methods:

Sending a Stop message from the context.  
Calling the stop method on a Phluxor\ActorSystem\Ref instance.  
Calling the stop method on a Phluxor\ActorSystem\ActorContext instance.  
