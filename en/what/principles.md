# Design principles

Phluxor is developed based on the following design principles. 

## Minimalistic, Not a Framework

Phluxor provides minimal functionality,  
offering the basic features of the actor model.  
It does not depend on any specific framework and is not a framework itself.  
Phluxor aims to offer the essential functions of the actor model  
without providing web application framework features or dependency injection (DI) containers.  
It strives to be small and easy to use.

## Standard

Phluxor uses proven technologies and adopts widely accepted concepts to build the actor model.  
It employs Protocol Buffers for serialization/deserialization and gRPC for clustering.  
Existing tools like Consul or Etcd are used for cluster management, and OpenTelemetry is used for actor monitoring.

## Message Passing

Communication between objects is done through message passing, and messages are guaranteed to be immutable.  
Messages must be serializable.  
Phluxor eliminates interdependencies between objects and avoids PHP-specific serialization mechanisms.
