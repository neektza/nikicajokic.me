---
layout: post
title: "Celluloid internals and the Actor model"
tags: concurrency ruby actors celluloid
comments: false
---

Continuing from previous two posts about [primitives and abstractions](link) in Ruby and the [reactor pattern](link), in the last part of the series we'll cover handpicked internals from the Celluloid library, which is a Ruby implementation of the Actor model, and explain the core ideas behind it.

## The Actor model

First, let's define what the *Actors* actually are.

> An actor is a computational entity that, in response to a message it receives, can concurrently:
> 
> * send a finite number of messages to other actors;
> * create a finite number of new actors;
> * designate the behavior to be used for the next message it receives.[^1]

What this Wikipedia paragraph is saying is that an Actor model is something inside of what computation is done only by sending messages. This is an ideal abstraction. Actual implementations of this model rarely limit themselves on doing computation only in this way. This is also the case in Celluloid.

The actor system is somewhat similar to an object system, the main difference being that in object based system everything done sequentially in contrast to actor based system where everything is done concurrently. In that regard, the actor model more accurately describes the real world because in the real world, everything happens concurrently.

In order to understand the about the Actor model implementation in Ruby we'll analyze several of the main classes where most of Celluloid's feature reside.

## Cell

Everything starts by wrapping the subject's class that's suppost to act as an actor in a `Cell`. Cell is basically a factory class that initializes the Actor, sets up its handlers and passes it allong to the CellProxy object, which it also initializes. The CellProxy object is used to convert the regular Ruby method protocol to inter-actor message protocol.

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
