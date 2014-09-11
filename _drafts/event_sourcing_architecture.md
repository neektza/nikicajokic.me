---
layout: post
title: "Event Sourcing or Append-only architecture"
tags: architecture
comments: false
---
	
## WcZg 2014 talk outline

- event sourcing intro/definition
	- wikipedia
	- "Using objects history to construct its state"
	- "Incrementaly building object state with a series of events"

- definition of an event
	- immutable / not updateable
	- has no identity, we're only interested in it's value
	- origninates elswhere?
	- brings new information
	- there has to be a system in place to prevent generating/storing multiple events with the same content?

- definition of an entity
	- has identity, we're able to update it
	- a product of our system (our domain)
	- incorporated (materialized) state of the system
	- has relationships to other things in the system

- simple example
	- simple diagram
	- simple code ?

- capabilities
	- complete rebuild of state
	- temporal query of past states
	- replay/reordering of incoming events if needed
	
- our system (intro)
	- problems we had
	- problems event-sourcing could potentially solve
	- our solution was organic
		- didn't know about event-sourcing unitl preparation of this talk

- our system (diagram in steps)
	- event comes in
	- it is processed, and entity is inited
	- event and entity are saved only if both are successful

- our system (epilogue/metrics)
	- easier to track bugs (isolate state changes)
	- able to produce better metrics

- other strong use cases
	- accounting
	- ...


## What?

Event Sourcing (or Append-only) architecture is a name for one of the Patterns of Enterprise Application Architecture that deals with consistency and temporality of application state.

> The fundamental idea of Event Sourcing is that of ensuring every change to the state of an application is captured in an event object, and that these event objects are themselves stored in the sequence they were applied for the same lifetime as the application state itself.[^1]

The above quote captures the fundamental idea of *Event Sourcing*, i.e. incrementaly building domain objects through a series of deltas that are recorded somewhere.

# Why?

Cases for storing state in such a way can vary, but benefits of such an architecture are many.

One of those benefits is the ability to perform temporal queries - e.g. to see the state of data as it was in the past. One example of a such a system is *git* - a version control system we all (hopefuly) use. 

Another yet is the ability to completely rebuild the application state out of past events. Such an event should happen rarely, if ever, but it is comforting to know that one can rebuild the current application state if anything happens to it.

A closely related benefit is the ability to handle lost or delayed messages. If a message that was sent from another part of the system (or a 3rd party system) never arrives, we can continue to handle other inconming events, and then, when the lost event does come, we can reverse events that came after it, apply it, and then apply all later events. In such a way we can handle communication problems that occur in systems that communicate via asynchronous messaging.

# Example







---
[^1]: Martin Fowler, [Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html)
