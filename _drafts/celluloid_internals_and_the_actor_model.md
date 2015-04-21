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

## Transforming objects into actors

Once the `Celluloid` module is included in a class, an included hook gets called and bunch of things happen to that class.

{% highlight ruby %}

 included(klass)

{% endhighlight %}

One of the first things that happens is that the class gets inheritable class-level properties. Normal Ruby class don't have that feature. If you're not familiar with what I'm talking about here, read more about it in my blog post about [class-level inheritable properties](http://pltconfusion.dev/2015/04/15/class_level_inheritable_properties).

Configuration stuff that happens next

1. The class gets a `mailbox_class` property, which in the default case is a `Mailbox`. An actor uses the mailbox, as expected, to receive messages from other actors. Since we need to bypass Ruby's regular inter-object messaging, we need a mailbox to store incoming messages. We'll cover the Mailbox class in detail a bit later.

2. The class gets a `proxy_class` property. Proxy objects are the ones that get returned by actor constructors and the ones that act as an interface to the underlying object by intercepting all calls to that object. This is where the bypassing of Ruby's messaging protocol actually happens, since the objects never receives messages directly.

3. The class gets `task_class` property. Tasks are used to resume and suspend method execution and by default Ruby's fibers are used to represent tasks. This is what enables code written using Celluloid to look like synchronous code, even though asynchronous operations are happening under the hood.

These three classes make up the foundation of Celluloid. MALO VISE OVDJE

After the subject class has been configured, we're ready to start creating actors.

## Cell

Everything starts by wrapping the subject's class in a `Cell`. Cell is basically a factory class that initializes the Actor, sets up its handlers and passes it allong to the CellProxy object, which it also initializes. The CellProxy object is used to convert the regular Ruby method invocation protocol to inter-actor message protocol.


## Proxy

## Task

## Mailbox

## ActorSystem

---
[^1]: Wikipedia
[^2]: allocate doc in rubydoc
