---
layout: post
title: "Celluloid internals and the Actor model"
tags: concurrency ruby eventmachine celluloid
comments: false
---

Continuing from previous two posts about [primitives and abstractions](link), and the [reactor pattern](link), in the last part of the series we'll cover handpicked internals of Celluloid and EventMachine, and explain the gist of their ideas.

## The Actor model

> An actor is a computational entity that, in response to a message it receives, can concurrently:
> 
> * send a finite number of messages to other actors;
> * create a finite number of new actors;
> * designate the behavior to be used for the next message it receives.[^1]

What this wikipedia paragraph is saying is that an Actor model is something inside of what computation is done only by sending messages. This is an ideal abstraction. Actual implementations of this model rarely limit themselves on doing compution only in this way. This is also the case in Celluloid.

The actor system is similar to the object system, the main difference being that in object based system everything done sequentially in contrast to actor based system where everything is done concurrently 

In order to understand the whole thruth about the Actor model implementation in Ruby we'll analyze several of the main classes where most of the functionality is implemented.

## Cell

Everything starts by wrapping the subject's class that's suppost to act as an actor in a ```Cell```. Cell is basically a factory class that initializes the Actor, sets up its handlers and passes it allong to the CellProxy object, which it also initializes. The CellProxy object is used to convert the regular Ruby method protocol to inter-actor message protocol.

The CellProxy inherits from the SyncProxy class which defines the protocol for sending synchronous calls to an actor

allocating[^2]

Everything 

## Proxy

## Task

## Mailbox

## ActorSystem

---
[^1]: Wikipedia
[^2]: allocate doc in rubydoc
