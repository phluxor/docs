# introduction

This guide provides a brief explanation of the actor model before explaining how to use Phluxor.

In many traditional programming models of PHP,  
programs are executed in a single thread and processed synchronously.  

Many programming models,  
such as the request-response model and event-driven model, assume this synchronous processing.  

However, modern web applications and distributed systems need to handle multiple requests and events simultaneously.  
As the times change, with cloud computing and microservices architecture,  
the processing models of applications are also evolving.

To address this situation, the actor model has gained attention.  
The actor model is one of the models for concurrent processing.

In the actor model, communication with other actors is done by sending messages,  
and receiving messages from other actors constitutes communication.

The actor model processes using the unit called an actor, and each actor operates independently.  
Therefore, it is characterized by its ease in handling requirements such as load balancing and scalability.

In the past,  
it was difficult for PHP applications to handle such requirements,  
but with Phluxor, you can easily implement the actor model in PHP and flexibly change the processing model of your application.

Like many actor model toolkits, Phluxor provides features such as actor creation,  
message sending and receiving, and actor monitoring.

Based on the philosophy of "let it crash,"  
actors are designed to handle errors themselves without affecting other actors.

It is based on concepts similar to actor model toolkits  
like [Akka](https://akka.io/), [Pekko](https://pekko.apache.org/), and [Proto Actor](https://proto.actor/),  
so those who have used these toolkits will quickly understand how to use it.

Conversely, those who have not used such toolkits can learn the actor model by using Phluxor  
and acquire knowledge that can be applied when using other toolkits.

![Actors](/images/actors.png "Actors")

Actors have a hierarchical structure and can send messages from parent actors to child actors.  
All actors have a root actor as their parent actor and communicate with other actors by sending messages.  
These actors all have addresses, and messages are sent to other actors using these addresses.

The familiar `return` statement in many languages, including PHP, is not used.  
Experience the new world of the actor model, which is different from the conventional programming paradigm.  

[install >](/en/guide/install.html)
